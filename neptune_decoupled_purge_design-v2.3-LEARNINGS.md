# Decoupled Purge on Amazon Neptune: Engineering Learnings

Nov 20, 2026 · Harish Kumar

A proof of concept that purges soft-deleted edges from Amazon Neptune without hurting the live cluster worked end to end, but only after eight defects that tests alone could not reveal. This document records what was built, what broke, and the rules and patterns worth reusing. It needs no access to the original code: every pattern is described so it can be rebuilt.

## The approach

Discovery and deletion are decoupled so the expensive scan never competes with production traffic.

1. **Discover on a clone.** A copy-on-write clone of the cluster is restored from the latest restorable time and given one instance. A read-only worker scans it for edges that are soft-deleted for a tenant and writes their IDs to S3 in compressed chunks of 5,000, checkpointing a cursor after each chunk. A manifest is sealed with a marker object, and only then are chunk records and queue pointers created. The clone is always deleted afterwards.
2. **Execute on the live writer.** A worker drains a per-tenant SQS queue of chunk pointers and processes each chunk in micro-batches of a few hundred IDs. A controller reads writer health and adjusts the rate (details below). Progress lives in DynamoDB.
3. **Orchestrate with Step Functions.** One state machine maps over enabled tenants. Per tenant it checks for a paused run, restores the clone, polls until it is ready, runs discovery as a Batch job on Fargate Spot, tears the clone down, then runs execution as a Batch job on Fargate On-Demand.

| Component | Role |
| --- | --- |
| Step Functions (Standard) | Per-tenant flow, clone lifecycle, Batch submission |
| AWS Batch on Fargate | Spot for discovery (interruptible, checkpointed), On-Demand for execution |
| SQS, one queue per tenant plus a shared DLQ | Chunk pointers only, 14-day retention, 3 receives before the DLQ |
| DynamoDB | Tenant config, run ledger, chunk ledger, cluster lease |
| S3 | Chunk manifests and the seal marker |
| CloudWatch | Writer metrics in, embedded-metric-format counters out |

## Result

Two test clusters (Serverless v2 and provisioned) each held 57,645 edges and 3,749 vertices. Per cluster, 2,883 edges were soft-deleted for the target tenant and 2,883 for a different tenant. The encryption posture was provider defaults only, with no customer-managed keys.

| | before | after |
| --- | --- | --- |
| edges | 57,645 | 54,762 |
| vertices | 3,749 | 3,749 |
| target tenant soft-deleted edges | 2,883 | 0 |
| other tenant's soft-deleted edges (must survive) | 2,883 | 2,883 |

The run ledger recorded 2,883 confirmed deleted and 0 absent, no-longer-eligible, uncertain or failed, so deleted plus absent plus no-longer-eligible equalled discovered. The DLQ was empty and no clone clusters were left behind.

## Defects that only live runs exposed

All unit tests and infrastructure mock tests passed before the first deployment. Eight defects still appeared against real AWS, and the shared-queue one would have silently corrupted a purge.

| # | Symptom | Root cause | Fix |
| --- | --- | --- | --- |
| 1 | Neptune rejected every request with a signature mismatch | The signed `content-type` was `application/json`, but the HTTP client sent `application/json; charset=utf-8` | Set the header explicitly with no charset |
| 2 | Bulk-loader status polling failed the signature check | SigV4 needs the canonical query string sorted by key; the request sent `errors=…&details=…` | Send `details=true&errors=true` |
| 3 | Worker crashed on exit | A process-exit handler cancelled an already-disposed cancellation token | Guard the cancel in try/catch |
| 4 | Test-data tagging failed with a cardinality error | Neptune edge properties do not accept a cardinality argument | Use plain `property(key, value)` on edges |
| 5 | A Step Functions state failed with `States.Runtime` | A Pass state replaced its input and dropped a field a later state needed | Pass the field through; add a static data-flow check |
| 6 | One cluster stayed unpurged while its ledger said `COMPLETED` | Both tenants shared one queue, so one tenant's worker consumed the other's chunk and applied it to the wrong writer | One queue per tenant, the queue URL stored in tenant config, and the worker refuses any pointer from another run |
| 7 | Discovery crashed when there was nothing to purge | The S3 SDK returns a null object list, not an empty one, for an empty prefix | Treat a null list as empty |
| 8 | A second tenant's clone was orphaned after the first tenant failed | A failing Map iteration cancels its siblings mid-teardown | `ToleratedFailurePercentage: 100`, with failures recorded as `FAILED` in the run ledger |

Defect 6 is the one to remember. The first purge reported success for both tenants while one cluster still held all 2,883 soft-deleted edges. Only a verification step that counts edges directly in the graph exposed it. Receive-and-return was rejected as a fix because every foreign receive would count toward `maxReceiveCount` and push valid chunks into the DLQ.

## AWS and Step Functions gotchas

Most of these cost a full 45-minute run to discover, so they are worth knowing up front.

- **SDK integration casing is not uniform.** Request parameters follow the Java v2 member names (`DbClusterIdentifier`, `DbInstanceClass`, `DbSubnetGroupName`), but `SourceDBClusterIdentifier` and `EnableIAMDatabaseAuthentication` keep their API casing. The `ValidateStateMachineDefinition` API catches wrong request names without any deployed resource.
- **Response fields** use `DbClusters`, `DbInstances` and `DbInstanceStatus`, as the Java v2 rule predicts.
- **`States.Runtime` cannot be caught.** A missing `$.Key` fails the whole execution and skips every `Catch`, leaving a billable clone running. Check data flow statically before deploying.
- **A failing Map iteration cancels its siblings.** Without `ToleratedFailurePercentage`, one tenant's failure aborts another tenant's teardown.
- **Neptune has no `.sync` integration.** Every wait is an explicit Describe polling loop. Detect "gone" with a describe filtered by `db-cluster-id` that returns an empty list, rather than by catching a not-found error name.
- **The HTTPS `/gremlin` endpoint has no query bindings.** Validate IDs and tenant names against a strict allow-list before writing them into query text.
- **SigV4:** sign exactly the `content-type` you send, and sort query-string keys. The AWS SDK signer avoids both traps, at the cost of a dependency.
- **Edge properties take no cardinality argument** in Gremlin on Neptune.
- **Neptune instance deletes are slow**, and a cluster can only be deleted after its last instance is fully gone.

## Timing and cost

The purge itself takes seconds. The ephemeral clone accounts for nearly all of a run's wall-clock time and cost.

| Step | Observed duration |
| --- | --- |
| Infrastructure deployment, 66 resources | about 11.5 min |
| Clone cluster and instance becoming available | roughly 20 to 35 min |
| Discovery of 2,883 edge IDs on a clone | 3.6 s |
| Execution of one 2,883-edge chunk | under 1 min of job time |
| Clone instance deletion, then cluster deletion | roughly 15 to 30 min |
| One complete purge execution | about 45 to 50 min |

- **Running cost** was roughly $0.4 an hour with both test clusters up. That is an estimate, not a billed figure. Each execution adds a clone per tenant for most of its length, so a failed run that leaves a clone behind costs real money.
- **Orphaned clones need a reaper.** Two runs left clones that had to be deleted by hand. A reaper should delete only clones that carry the purge tags, whose owning execution is no longer running, and whose name matches the clone pattern.
- **Verify against the graph, not the ledger.** A verification step that counts edges and soft-deleted edges directly was the only check that exposed defect 6.
- **Stale state after a cancelled run** (a `RUNNING` ledger row and a leftover queue message) must be cleared before re-running.

## Design rules worth keeping

Each rule below was enforced by code or an automated test in the proof of concept, and each earned its place.

**Data safety**

1. Delete edges only. IAM cannot scope `neptune-db:DeleteDataViaQuery` to edges, so the application must reject any rule whose entity type is not `EDGE`. Never add a vertex path.
2. Every tenant ID and edge ID written into Gremlin text must match `^[A-Za-z0-9_\-:.@/]{1,256}$`. A value outside the allow-list is rejected, never escaped and accepted.
3. The drop always repeats the guard in the same traversal, with a boolean literal (the property is case-sensitive and typed): `g.E('e1','e2').has('tenantId','tenant-001').has('isDeleted',true).drop()`.
4. `drop()` returns no count. Every number comes from classify reads (which IDs exist, which are still eligible) and a confirm re-read of the eligible IDs. Never report a delete from the drop response.
5. Per micro-batch, deleted + already-absent + no-longer-eligible must equal the batch size. A batch that ends uncertain or failed is not checkpointed: the chunk is released as `UNCERTAIN` and re-classified later, which is safe because classification is idempotent.

**Queue and ledger correctness**

6. Use one chunk queue per tenant, and have the worker refuse any pointer whose run key is not its own. A shared queue lets one tenant's worker apply another tenant's chunk to the wrong writer.
7. Seal the manifest (write a marker object, then set a sealed flag) before creating chunk records or sending pointers. Workers never run on an unsealed manifest.
8. Claim a chunk with a conditional update: status `PENDING`, or status `CLAIMED` or `UNCERTAIN` with an absent or expired lease. Increment a lease-version counter in the same update; it is the fencing token.
9. Condition every checkpoint and release on `LeaseVersion = :v AND Status = CLAIMED`. Complete a chunk in one `TransactWriteItems`: the chunk update (`Status <> COMPLETED AND LeaseVersion = :v`) plus an `ADD` of its counts into the run ledger.
10. Delete the SQS message only after that transaction commits. Set `maxReceiveCount` to 3 and keep the DLQ retention at least as long as the source queue's 14 days.

**Operations**

11. Discovery connects only to the clone endpoint, and its role allows `neptune-db:ReadDataViaQuery` and `GetQueryStatus` only. Use an integer offset as the cursor so nothing sensitive reaches stores that use a platform key.
12. Always tear the clone down, on success and on every `Catch`: delete the instance, poll until a `db-cluster-id` filtered describe is empty, delete the cluster, poll again. Start execution only after both are gone.
13. Treat a missing CPU datapoint as Red, a missing queue-depth or undo-log datapoint as 0, and a changed writer instance ID as Red. Ignore Serverless-only signals on provisioned clusters.
14. Allow one worker per source writer, using a DynamoDB lease with a fencing token and heartbeat. Exit with a distinct code for "lease busy or lost" and let the Batch retry strategy handle it.
15. Every `$.Key` referenced in the state machine must be produced on every upstream path. `States.Runtime` cannot be caught, and a miss leaks a billable clone.
16. Build images for `linux/amd64`, because Fargate Spot is x86_64 only. Use immutable ECR tags, so every code change needs a new tag.
17. For a low-cost proof of concept, use provider-default encryption only: Neptune `storageEncrypted` with the AWS-managed key, S3 SSE-S3, SQS-managed SSE and DynamoDB default encryption. Leave `kmsKeyId` unset.

## Reference patterns

**SigV4 for Neptune** (service `neptune-db`, port 8182). Both live signature failures came from details here.

```text
canonical request =
  METHOD \n path \n query \n
  headers (lowercase, sorted by name, each "name:value\n") \n
  signedHeaders \n sha256hex(body)
rules:
  - query keys sorted:   details=true&errors=true        (not errors=...&details=...)
  - signed content-type == sent content-type exactly:    application/json   (no "; charset=utf-8")
  - signed headers: host (includes :8182), x-amz-date, content-type (POST only), x-amz-security-token (when present)
  - scope = yyyyMMdd/<region>/neptune-db/aws4_request
```

**Gremlin shapes.** Discovery pages by offset on a static clone; the other four run on the live writer.

```text
discover  g.E().has('tenantId',T).has('isDeleted',true).id().order().range(off, off+5000)
present   g.E('a','b').id()
eligible  g.E('a','b').has('tenantId',T).has('isDeleted',true).id()
drop      g.E('a','b').has('tenantId',T).has('isDeleted',true).drop()
count     g.E().has('tenantId',T).has('isDeleted',true).count()
```

Edge properties take no cardinality, so tagging is `.property('tenantId',T)`, never `.property(single, ...)`. GraphSON 3 answers arrive typed (`{"@type":"g:List","@value":[...]}`) and must be unwrapped.

**Chunk claim** (illustrative attribute names):

```text
ConditionExpression: attribute_exists(RunKey) AND (#s = :pending OR (#s IN (:claimed,:uncertain)
                     AND (attribute_not_exists(LeaseExpiresAt) OR LeaseExpiresAt < :now)))
UpdateExpression:    SET #s = :claimed, ClaimedBy = :o, LeaseExpiresAt = :exp ADD LeaseVersion :one
Checkpoint condition: LeaseVersion = :v AND #s = :claimed
```

**Throttle thresholds** (provisioned defaults; Serverless lowers the batch to 100/350 and the max rate to 3.0):

| Signal | Green | Yellow | Red |
| --- | --- | --- | --- |
| CPU % | below 45 | 45 to 69.9 | 70 or more, or missing |
| Queue depth (pending, or accepted minus running) | 0 | 1 to 5 | above 5 |
| `UndoLogListSize` | below 250,000 | 250,000 to 1,000,000 | above 1,000,000 |
| Conflict retries per minute | 0 | 1 to 3 | above 3 |
| App p95 vs 250 ms SLO | below 80% | 80% to 99% | 100% or more |
| Serverless `NCUUtilization` % | below 50 | 50 to 75 | above 75 |
| Serverless `BufferCacheHitRatio` % | 99 or more | 95 to 99 | below 95 |

AIMD control: three consecutive Greens add 0.25 to the rate (max 4.0) and grow the batch by at least 10. Yellow halves the rate (floor 0.25) and the batch. Red pauses, drops the rate to the floor, extends the SQS visibility timeout and waits for fresh Greens. Starting values: batch 200, max batch 500, rate 1.0 batches per second.

**Step Functions Neptune calls.** Request parameters accepted by `ValidateStateMachineDefinition`:

```text
restoreDBClusterToPointInTime  DbClusterIdentifier, SourceDBClusterIdentifier, RestoreType=copy-on-write,
                               UseLatestRestorableTime=true, DbSubnetGroupName, VpcSecurityGroupIds[],
                               EnableIAMDatabaseAuthentication, DbClusterParameterGroupName, Tags[],
                               ServerlessV2ScalingConfiguration{MinCapacity,MaxCapacity}  (Serverless only)
createDBInstance               DbInstanceIdentifier, DbInstanceClass ('db.serverless' or a class), Engine=neptune, DbClusterIdentifier
deleteDBInstance / Cluster     DbInstanceIdentifier | DbClusterIdentifier, SkipFinalSnapshot=true
gone check                     describeDBInstances / describeDBClusters with Filters=[{Name:db-cluster-id, Values.$:States.Array($.CloneId)}]
                               then States.ArrayLength(...) == 0
response fields                $.DbClusters[0].Status | .Endpoint | .LatestRestorableTime ; $.DbInstances[0].DbInstanceStatus
```

A state that replaces its input with `Parameters` and no `ResultPath` must re-list every key later states need (clone ID, run ID, queue URL and similar). Set `ToleratedFailurePercentage: 100` on the per-tenant Map. Preflight must fail closed: if the source has no `LatestRestorableTime`, skip the run instead of falling back to a full-copy restore.

## Components and their contracts

A single container image can serve every role, with the command selecting the behaviour and all inputs passed as environment variables.

| Component | Responsibility |
| --- | --- |
| Neptune client | SigV4-signed HTTPS calls to `/gremlin`, `/gremlin/status` and `/loader`; unwraps GraphSON; exposes flags for concurrent-modification, read-only-violation and throttling errors |
| Query builder | Owns every Gremlin string; validates identifiers against the allow-list; rejects non-edge rules at construction |
| Discovery worker | Pages the clone, writes compressed chunks, checkpoints the cursor, seals the manifest, then creates chunk records and queue pointers |
| Executor | Claims chunks, runs micro-batches, honours the maintenance window, completes or releases each chunk, finishes the run as `COMPLETED` or `PAUSED` |
| Micro-batch processor | Classify, conditional drop, confirm by re-read; returns outcome counts and whether a retry is needed |
| Backpressure classifier and rate controller | Turns a writer snapshot into Green, Yellow or Red and adjusts rate and batch size |
| Writer monitor | Resolves the current writer, reads CloudWatch every 60 s and `/gremlin/status` every 15 s |
| Cluster lease | One worker per source writer, with a fencing token and a 20 s heartbeat |
| Ledgers | Run ledger (status, discovery and execution counters) and chunk ledger (claim, checkpoint, complete, release) |
| Seed tool | Bulk-loads sample data, tags edges for the test scenario, writes the tenant config |

Inputs per command (names illustrative):

| Command | Inputs | Touches |
| --- | --- | --- |
| `discover` | tenant ID, run ID, queue URL, clone endpoint | The clone only, read-only |
| `execute` | tenant ID, run ID, queue URL, resume flag | The live writer named in the tenant config |
| `seed` | tenant ID, compute mode, cluster identifiers, writer endpoint, data bucket, loader role, queue URL | The live writer; upserts the tenant config |
| `verify` | tenant ID | The live writer, read-only counts |

Exit codes: `0` success (a paused run also exits 0), `75` lease busy or lost (retried by the Batch job definition), `1` error, `143` cancelled. The queue URL is not baked into the job definition: the state machine injects each tenant's own queue from its config item.

## How to verify

**Automated tests worth writing**

| Area | What to assert |
| --- | --- |
| Throttle bands | Each threshold, missing-data rules, writer failover, Serverless-only signals ignored on provisioned |
| Rate control | Three Greens raise the rate; Yellow halves it; Red pauses; rate and batch stay within bounds |
| Query safety | Non-edge rule rejected, hostile IDs rejected rather than escaped, drop text carries the guard |
| Micro-batch | Four outcomes classified, reconciliation holds, replay reports absent, conflict retries back off, a lost drop response is resolved by re-read, a read-only violation leaves the batch retryable |
| Run isolation | A pointer from another run is refused; the queue URL survives a config round-trip |
| Infrastructure | No KMS resources, one queue per tenant with redrive 3 and long-enough DLQ retention, Spot for discovery and On-Demand for execution, x86_64 job definitions, security groups admit port 8182 only from the intended source, discovery role read-only, executor scoped to production cluster IDs, delete permissions limited to clone names |
| State machine | Every transition target exists, and every `$.Key` is produced on all upstream paths |

Use an in-memory graph that interprets the engine's own query shapes for the micro-batch tests, so classify, drop and confirm run without a cluster.

**Live verification.** Compare the graph before and after: total edges, vertices, the target tenant's soft-deleted edges, and a different tenant's soft-deleted edges. Expect the target tenant's count to reach zero, the other tenant's to stay unchanged, and vertices to be untouched. Then check the run ledger counters, an empty DLQ, and no leftover clone clusters.

**Changing the state machine.** Run the static data-flow check, then call `ValidateStateMachineDefinition` with type `STANDARD` on the generated definition. The first catches missing fields; the second catches wrong request parameter names. Neither needs a deployed resource.

**Suggested end-to-end sequence.** Run the unit and infrastructure tests, deploy the infrastructure, build the `linux/amd64` image with a new immutable tag and update the stack to use it, seed both clusters, start one execution, verify against the graph, and destroy the stack when finished. Expect about 11.5 minutes to deploy and 45 to 50 minutes per purge.

## Gaps and next steps

The core flow is proven, but five things stand between a proof of concept and production.

1. **Add an orphan clone reaper.** Live runs left clones twice. Delete a clone only when it carries the purge tags, its owning execution is not running (or its lease heartbeat has expired), and its name matches the clone pattern. Age alone is never enough.
2. **Surface tenant failures.** With `ToleratedFailurePercentage: 100` an execution can succeed while a tenant's run is `FAILED` in the ledger. Add an alarm on the run ledger, or a final Choice state that fails the execution.
3. **Replace offset paging in discovery.** It is O(n²) at tens of millions of edges. Page by an ID cursor instead, after confirming that `hasId(gt(...))` behaves on Neptune; this was not tested.
4. **Share rate between tenants on one cluster.** The proof of concept lets one worker hold a cluster lease while others wait. A shared AIMD rate across tenants is not implemented, and no application-latency signal is wired in.
5. **Paginate the tenant scan.** The state machine reads a single DynamoDB scan page, which breaks above 1 MB of tenant configuration.

Also left out: audit reporting with Athena and Glue, customer-managed or bring-your-own keys, and tenant-tagged STS sessions. Per-tenant queue creation belongs in a tenant onboarding step rather than the core stack.
