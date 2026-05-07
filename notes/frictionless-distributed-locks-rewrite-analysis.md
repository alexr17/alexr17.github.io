---
title: "Rewrite analysis: conditional writes blog"
status: raw-analysis
source_note: notes/frictionless-distributed-locks-for-the-data-lakehouse.md
---

# Rewrite analysis: conditional writes blog

## Intent

Rewrite the current company-style article into a personal engineering blog.

The new framing should start from personal experience:

> When I was working at Onehouse, I implemented a distributed locking service using conditional writes in object storage for Apache Hudi.

Then pivot to the broader ecosystem:

> Conditional writes in object storage look like a small API feature, but they unlock a surprising amount of system design. They are also much more fickle than the happy-path examples imply.

The post should include the original Hudi design as a concrete case study, then use turbopuffer, Gunnar Morling, Martin Kleppmann, AWS SDK issue #6580, GCS limits, and DigitalOcean Spaces compatibility as evidence that the sharp edges are real across the ecosystem.

## Proposed thesis

Conditional writes turn object storage from "just blob storage" into a basic coordination primitive. That is powerful because object storage is already durable, available, cheap, and operationally ubiquitous. But the primitive is not a lock service by itself. Once retries, leases, provider-specific limits, S3-compatible implementations, and fencing semantics enter the picture, correctness depends on engineering the full protocol around the primitive.

Short version:

> Conditional writes are a great primitive, not a complete distributed systems product.

## Tone and positioning

Use a first-person, practical tone. This should not read like product marketing for Hudi or Onehouse.

Avoid:

- "you will learn"
- broad lakehouse adoption claims
- company-blog phrases like "industry-proven durability and availability guarantees"
- presenting the design as solved and generally safe

Prefer:

- "I implemented..."
- "The API looked simple..."
- "The first production surprise was..."
- "This is the part the toy examples skip..."
- "The right lesson is not 'never do this'; it is 'know exactly what contract you are building on.'"

## Candidate titles

- Conditional Writes Are a Distributed Systems Primitive, Not a Lock Service
- The Sharp Edges of Object Storage Conditional Writes
- Building Locks on Object Storage Is Possible. It Is Also Fickle.
- What I Learned Building Distributed Locks on S3 and GCS

Recommendation: use "The Sharp Edges of Object Storage Conditional Writes". It is broad enough for the ecosystem angle and personal enough for the production story.

## Links to include

- Original Hudi RFC: https://github.com/apache/hudi/blob/master/rfc/rfc-91/rfc-91.md
- Hudi retry-disable PR: https://github.com/apache/hudi/pull/17869
- AWS SDK issue #6580: https://github.com/aws/aws-sdk-java-v2/issues/6580
- Martin Kleppmann on distributed locks: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Gunnar Morling on leader election with S3 conditional writes: https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/
- HN discussion of Gunnar's post: https://news.ycombinator.com/item?id=41357123
- turbopuffer LinkedIn post: https://www.linkedin.com/posts/turbopuffer_s3s-conditional-writes-were-the-key-ingredient-activity-7346587427584491520-HaD0/
- turbopuffer first-principles database podcast/blog: https://turbopuffer.com/blog/podcast-database-from-first-principles
- turbopuffer object-storage queue post: https://turbopuffer.com/blog/object-storage-queue
- turbopuffer conditional write docs: https://turbopuffer.com/docs/write#conditional-writes
- GCS quotas and same-object write limit: https://cloud.google.com/storage/quotas
- DigitalOcean Spaces S3 compatibility: https://docs.digitalocean.com/products/spaces/reference/s3-compatibility/
- DigitalOcean Spaces API reference: https://docs.digitalocean.com/reference/api/spaces/

## Source notes

### Hudi RFC and Onehouse story

The RFC is the personal anchor. It explicitly lists `@alexr17` as proposer and describes "Storage-based lock provider using conditional writes." The abstract says Hudi previously relied on external lock systems like ZooKeeper and proposes using conditional writes in GCS and S3 to coordinate concurrent writes.

Important details from the RFC:

- Single lock file per table: `.hoodie/.locks/table_lock.json`.
- Lock file fields: `owner`, `expiration`, `expired`.
- `tryLock()` creates or overwrites the lock with preconditions.
- `renewLock()` extends expiration as a heartbeat.
- `unlock()` marks expired rather than deleting because S3 does not support conditional deletes.
- Cloud-specific implementations were necessary because the standard filesystem layers did not expose conditional writes consistently.
- Clock drift was explicitly scoped: 500 ms tolerance, writers should be in the same cloud provider.

Use this as: "Here is the link to the original RFC. It is a much drier design doc than this post."

### turbopuffer

The public LinkedIn post says S3 conditional writes were a key ingredient that made their architecture possible. The public page does not expose much more without sign-in, but it links to their docs.

turbopuffer docs expose conditional writes at the application API level:

- Supports conditional upserts, patches, and deletes.
- Conditions are evaluated against the current document.
- Examples include version checks and insert-if-not-exists.

Use this as evidence that conditional writes are not only a lakehouse/concurrency-control trick. They are showing up in vector/search/data APIs as a first-class product capability.

Do not overclaim turbopuffer internals from the LinkedIn post. Safe phrasing:

> turbopuffer has said publicly that S3 conditional writes were a key ingredient in making their architecture possible, and their public write API exposes conditional writes as a user-facing primitive.

The turbopuffer podcast transcript is a stronger source for the general ecosystem section. It walks through building a database on object storage from first principles:

- Start with a WAL prefix in object storage: `1.json`, `2.json`, `3.json`.
- Replay the WAL to answer queries, then add caches and derived indexes for performance.
- For strong reads, list the WAL prefix before serving a query so the node knows the latest committed WAL pointer.
- For multiple writers, one option is a separate consensus/lock service; another is object storage itself.
- The conditional-write shape is: list current WAL entries, decide the next file number, then write it with a condition that rejects the write if that object already exists.
- This avoids two writers successfully writing the same WAL slot, but it changes ordering semantics. The accepted order of writes may differ from request arrival order unless the system introduces a single writer, a sequencer, or another ordering mechanism.
- Alternative key layouts like UUIDv7 or writer IDs reduce conflicts but weaken or complicate ordering guarantees.
- Object-storage-backed databases have extra cost/performance dimensions beyond ordinary read/write/storage amplification: get amplification and put amplification.
- Batching or group commit can trade write latency for lower put amplification. The transcript gives examples around batching writes into time windows rather than committing every write independently.

This source supports a useful broadening move:

> Hudi used conditional writes to coordinate table commits. turbopuffer discusses the same primitive in the context of WAL append and multi-writer object-storage databases. In both cases the core problem is not "can I upload a blob?" It is "can I create a small, durable ordering or ownership fact exactly once?"

Potential contrast:

- Hudi lock provider centralizes contention on one lock object.
- turbopuffer's simplified WAL model spreads writes across append-only WAL object names.
- Both rely on put-if-absent semantics, but the hot-key and ordering tradeoffs differ.

The turbopuffer object-storage queue post is an even cleaner source for the "designing across object stores has ramifications" point.

Important points from that post:

- The queue starts as one `queue.json` file repeatedly overwritten with the full queue.
- Pushers and workers both use compare-and-set writes so each update only succeeds if `queue.json` has not changed since it was read.
- turbopuffer explicitly says this works well up to one request per second, and calls that out as a limit imposed by GCS.
- Group commit is introduced to decouple request rate from write rate.
- Even with group commit, multiple clients contending on one object can only fit roughly `1 / write_latency` writes per second, and GCS still has the one-RPS same-object limit.
- The production design moves object-storage writes behind a stateless broker so there are fewer writers contending over the object.
- CAS remains the correctness mechanism during failover: if two brokers briefly exist, one eventually sees CAS failure and backs off.
- The conclusion is useful for tone: object storage has few but powerful primitives, and the trick is learning the boundaries well enough to design inside them.

Use this to strengthen the GCS section:

> turbopuffer ran into the same shape of limitation from a different direction. A single JSON queue file on object storage is simple and correct with CAS, but GCS's one-write-per-second limit for the same object forces the design toward batching and brokered writes. That is the broader multi-cloud lesson: the hard part is not only finding a primitive that exists on S3, GCS, and Azure; it is designing around the different latency, throttling, and compatibility envelopes of each provider.

This also gives the Hudi story a cleaner bridge:

- Hudi used a single lock file and then hit same-object mutation limits in production-like paths.
- turbopuffer's queue post shows the same hot-object issue in a queue.
- Both examples suggest that single-object CAS is a good starting point, but high-rate systems need group commit, fewer writers, sharding, epoch files, or another design that reduces mutation pressure on one object.

### turbopuffer / Simon note on get-if-not-match

User-provided source note from Simon, CEO/founder of turbopuffer, with attached benchmark screenshot.

Text:

> get-if-not-match is important for building fast databases on object storage. used in e.g. tpuf for the WAL check to make sure the cache has the latest data.
>
> of the big 3 (Azure/S3/GCS), it may surprise many that Azure comes out the winner!
>
> (S3X is S3 one-zone, GCR is GCP's equivalent)

Screenshot summary:

Benchmark label: `GET 304 - 128KiB - 512 objects - ms`

| Provider | p50 | p99 | p999 |
| --- | ---: | ---: | ---: |
| S3X | 2 | 3 | 6 |
| AZURE | 3 | 9 | 23 |
| TIGRIS | 7 | 15 | 25 |
| S3 | 9 | 30 | 55 |
| GCS | 11 | 26 | 53 |
| GCR | 14 | 30 | 68 |
| R2 | 51 | 132 | 233 |

Why it matters for the rewrite:

- Conditional reads are part of the same design space as conditional writes.
- For object-storage-backed databases, "is my cache still fresh?" can be as important as "can I atomically claim this write?"
- `GET`/`HEAD` with `If-None-Match` can turn a freshness check into a cheap `304 Not Modified` response, avoiding unnecessary object transfer.
- This fits turbopuffer's WAL/cache story: before serving from cache, check whether the WAL pointer/object has changed.
- Provider latency differences matter directly because these freshness checks can sit on the read path.
- The post can mention this as a sibling primitive: conditional writes help establish ordering/ownership facts; conditional reads help cheaply validate cached state.

Use carefully:

- This is user-provided from a screenshot/post, not independently fetched from a public URL in this run.
- Do not over-index the blog on the exact benchmark table unless the final post links to the original public post or clearly says "Simon Eskildsen shared a benchmark..."
- Good sentence:

> Simon Eskildsen has also pointed out the read-side version of this story: `GET` with `If-None-Match` is important for object-storage databases because a `304 Not Modified` can tell a query node that its cached WAL state is still current without downloading the object again.

### Gunnar Morling

Gunnar's post is a clean independent explanation of leader election on S3 conditional writes.

Key points to borrow:

- S3 added `If-None-Match: *`, which returns `412 Precondition Failed` if the object already exists.
- He uses epoch-numbered lock files, e.g. `lock_0000000001.json`, instead of overwriting a single file.
- Listing determines the latest epoch.
- A new leader creates the next epoch file with `If-None-Match`.
- The post explicitly discusses validity/leases and the need to fence off stale leaders.

Use this as the "ecosystem independently converged on the same primitive" section.

Potential contrast with Hudi:

- Hudi used a single lock file plus ETag/generation preconditions for updates.
- Gunnar's design uses new lock files per epoch to work around S3's initially limited conditional write shape.
- Both designs are trying to get an atomic "only one writer wins" property out of object storage, but the shape of the object keys and metadata matters.

### Martin Kleppmann

Use as the correctness guardrail.

Important ideas:

- First clarify whether a lock protects efficiency or correctness.
- A lease can expire while the old holder is paused, then the old holder resumes and writes stale data.
- A lock service alone is not enough if the protected resource cannot reject stale work.
- Fencing tokens are the standard solution: each lock acquisition gets a monotonically increasing token, and the storage/resource rejects older tokens.
- Timing assumptions, process pauses, clock jumps, and network delays are where lock algorithms become subtle.

This should keep the blog from sounding like "conditional writes solve distributed locking." The nuanced position:

> Conditional writes can implement the atomic acquisition path, but the surrounding system still needs a story for leases, stale holders, and whether stale work can be fenced.

### Hacker News discussion

The HN thread around Gunnar's post reinforces the same caution: eventual leadership is not the same as safety, and systems must be prepared to detect or fence work from a previous leader.

Useful angle:

- Mention that this topic predictably attracts distributed-systems caveats.
- Do not use HN as primary authority; use it as evidence that practitioners immediately focus on fencing and timing assumptions.

### AWS SDK issue #6580

This issue exactly matches the production problem described in the current draft.

The reported failure mode:

1. Client sends `PutObject` with `If-None-Match: *`.
2. The first request reaches S3 and creates the object.
3. The response is lost before reaching the client.
4. The SDK retries.
5. The retry sees the object now exists and returns `412 Precondition Failed`.

From the application layer, a `412` usually means "someone else won." In this retry case, the caller may actually have won, but the retry hides that fact.

This is one of the central production lessons:

> A conditional write is not idempotent unless you make the request idempotent at a higher layer.

Potential phrasing:

> The failure mode is nasty because every individual component is behaving reasonably. S3 created the object once. The SDK retried after a network failure. The second conditional request correctly failed. The bug is in the semantic gap between transport retry and distributed protocol state.

### Hudi PR #17869

This is the concrete "we hit it too" artifact.

The PR title is "fix: disable retries in s3/gcs storage lock clients for storage based LP". It was merged February 5, 2026.

The PR summary says the storage lock provider should not use retries because the DFS layer may not handle SDK retries idempotently, and this was seen at least for AWS S3. It also notes S3/GCS failures would not be retried at the SDK/client layer, while lock renewal had its own retry loop.

This is a good pivot from the happy path into production reality:

> Our first reaction was to disable storage-client retries and rely on protocol-level retries where we understood the semantics.

Then add the new user-supplied lesson:

> That was also not a free lunch. It surfaced more transient errors, and on GCS it exposed the same-object write limit.

### GCS same-object write limit

Google Cloud Storage documents a maximum write rate to the same object name of one write per second. Writing above that rate can produce throttling errors.

This matters for a single-lock-file design. A heartbeat, release, reacquisition, and contention can concentrate writes on exactly one object: `table_lock.json`.

Possible lesson:

> The lock file is intentionally a hotspot. That is the point. But object stores are often optimized for high aggregate write throughput across keys, not rapid mutation of one key.

Design implication:

- Single-file CAS lock is simple and works well at moderate rates.
- If the protocol writes the same object too frequently, GCS can throttle.
- Epoch-file designs avoid overwriting the same object for acquisition but may add listing costs and cleanup complexity.
- Heartbeat interval and lease duration are not cosmetic configuration; they must respect provider limits.

### DigitalOcean Spaces

DigitalOcean Spaces is S3-compatible with partial S3 support.

Public docs say Spaces provides an S3-compatible API with partial support for Amazon S3 features. The API reference lists `If-Match` and `If-None-Match` for `GetObject`/`HeadObject`, but the `PutObject` supported headers list does not include `If-Match` or `If-None-Match`.

This supports the user's point that Spaces did not natively support conditional writes through the S3 SDK in the way Hudi needed.

Use carefully:

> One S3-compatible provider we tested, DigitalOcean Spaces, did not expose the `PutObject` conditional write behavior we needed through the S3 SDK. The broader lesson is that "S3-compatible" does not mean "compatible with every S3 concurrency primitive."

Do not make broader claims about all DigitalOcean behavior beyond the tested/observed limitation and current docs.

## Rewrite structure

### 1. Personal hook

Start with:

> When I was working at Onehouse, I implemented a distributed locking service for Apache Hudi using conditional writes in object storage.

Then immediately clarify:

> This was not a new consensus system. It was an attempt to remove an external dependency from a data lakehouse write path by using the atomic write preconditions that S3 and GCS already provide.

Link to RFC in the first or second paragraph.

### 2. Why conditional writes are exciting

Explain the primitive:

- Create if absent: `If-None-Match: *` or GCS `if_generation_match=0`.
- Overwrite only if version matches: `If-Match` / ETag or GCS generation match.
- This is enough to build compare-and-swap-like flows.

Broaden to ecosystem:

- Hudi uses it for locks.
- Gunnar shows leader election on S3.
- turbopuffer frames it as an architecture enabler and exposes conditional writes in its API.

### 3. The Hudi design in human terms

Keep this shorter than the current draft:

- One lock file per table.
- The winner writes the file with an owner and expiration.
- Heartbeat renews the lease.
- Unlock marks expired.
- Precondition failures mean someone else changed the lock state.

Avoid deep Hudi concurrency control taxonomy unless needed.

### 4. The distributed systems caveat

Bring in Kleppmann:

- Ask whether the lock is for efficiency or correctness.
- Leases can expire while old holders keep running.
- Fencing is the hard part.

Connect to Hudi:

- In Hudi, the lock guards timeline/commit operations; the surrounding protocol must still detect conflicts and fail safely.
- Do not imply object storage conditional writes magically solve stale writer problems.

### 5. The production bug: retries changed the meaning of 412

Tell the #6580 story as a concrete incident:

- We saw the same issue before/alongside the public AWS SDK report.
- A request can succeed remotely, lose its response, retry, then fail with 412.
- The client sees "lost lock race" even though it may have acquired the lock.

Make this a core anecdote, not a footnote.

### 6. The attempted fix: disable SDK retries

Use PR #17869:

- We disabled retries inside the S3/GCS storage lock clients.
- The reason: retries need protocol semantics, not generic transport semantics.
- Renewal already had its own retry loop where failures could be interpreted in the context of lock expiration and owner state.

Then add the cost:

- Disabling lower-level retries exposed more transient failures.
- It also exposed the GCS same-object write limit.

### 7. Provider-specific traps

Subsections:

- S3: conditional write semantics plus retry/idempotency trap.
- GCS: strong generation preconditions, but one write per second to the same object name.
- S3-compatible services: DigitalOcean Spaces showed that `S3-compatible` may omit the exact conditional write support required for a lock protocol.

This is the section that makes the post general and useful.

### 8. Engineering guidance

Concrete lessons:

- Treat conditional writes as a protocol primitive.
- Disable or control SDK retries around non-idempotent conditional writes unless you add idempotency tokens or can reconcile state.
- Prefer protocol-level retries where the code knows whether a failed write was acquisition, renewal, or release.
- Design key layout with provider limits in mind: single hot object vs epoch files.
- Make lease and heartbeat settings respect provider write limits.
- Test with injected lost responses, retries, throttling, 412/409 responses, and S3-compatible providers.
- Document which object stores are actually supported; do not hide behind "S3-compatible."
- If correctness depends on the lock, define the fencing/conflict-detection story explicitly.

### 9. Conclusion

End with:

> I still like conditional writes. They let you build useful coordination protocols on infrastructure you already operate. But the primitive is sharp. You need to engineer the retry semantics, provider limits, compatibility matrix, and fencing story as part of the design, not as cleanup after the happy path works.

## Possible outline

1. **The API that looked too small to matter**
2. **The lock I built at Onehouse**
3. **Why everyone got excited about conditional writes**
4. **A lock acquisition is not a lock protocol**
5. **The retry bug that made 412 ambiguous**
6. **Why disabling retries was only half a fix**
7. **GCS, Spaces, and the myth of one object storage API**
8. **Rules I would use next time**

## Raw claims to verify before final rewrite

- Exact sequence and timing of Hudi production incidents.
- Whether the #6580 public issue was filed after Hudi hit the issue; phrase as "same failure mode" unless timing is confirmed.
- Exact GCS error codes observed when the one-write-per-second limit hit.
- Whether DigitalOcean Spaces failed on `If-None-Match`, `If-Match`, or both, and what response/error it returned.
- Whether Onehouse can be named in relation to production incidents and customers.
- Whether turbopuffer's LinkedIn post has more detail behind sign-in that should be quoted or paraphrased.
