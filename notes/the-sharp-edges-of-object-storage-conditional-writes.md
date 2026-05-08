---
layout: note
title: "What I Learned about Building Distributed Locks on S3 and GCS"
date: 2026-04-01
---

When I was working at Onehouse, I implemented a distributed locking service for Apache Hudi's optimistic concurrency control using conditional writes in object storage.

I started the project about two months into Onehouse, after mostly building backend APIs for MySQL at Microsoft. I had barely worked with S3 before. The idea sounded simple: instead of requiring a separate lock service like ZooKeeper or DynamoDB, use the atomic write preconditions already available in cloud object stores.

I wrote the original Hudi design up in [RFC-91: Storage-based lock provider using conditional writes](https://github.com/apache/hudi/blob/master/rfc/rfc-91/rfc-91.md). The RFC is the dry version. This is the version about what I learned about S3, GCS, conditional writes, and the awkward places where a small storage API turns into distributed-systems work.

## The Tiny API

The core primitive is compare-and-set for object storage.

For a long time, S3 was the awkward missing piece here. GCS and Azure had supported object preconditions for years, but S3 did not support the create-if-absent write path that many of these designs want. That changed in August 2024, when [S3 added conditional writes for `PutObject` and `CompleteMultipartUpload`](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes/).

On S3, `If-None-Match: *` says "create this object only if it does not already exist." `If-Match` says "overwrite this object only if its current ETag is the one I observed." GCS exposes the same idea through generation preconditions, including `if_generation_match=0` for create-if-absent. Azure Blob Storage has its own version of conditional request headers.

That is enough to build useful coordination protocols:

- only one writer creates a file for a given logical slot
- only one process updates a lock file from the version it observed
- a reader can cheaply ask whether cached data is still current

That small set of operations has started showing up in a bunch of systems that want coordination without running a separate coordination service.

## Recent Examples

turbopuffer has talked publicly about S3 conditional writes being important to its architecture. Their [first-principles object-storage database discussion](https://turbopuffer.com/blog/podcast-database-from-first-principles) gets at the same underlying problem: when multiple writers append to a WAL on object storage, how do you make sure exactly one writer claims the next durable slot?

Terraform is another good example. In Terraform 1.10, the S3 backend added optional native state locking with a lock file written using `If-None-Match`, removing the need for DynamoDB in that path. Bruno Schaatsbergen's writeup on [S3 native state locking](https://www.bschaatsbergen.com/s3-native-state-locking) shows the same tradeoff from a very different ecosystem: the primitive removes infrastructure, but the implementation still has to care about migration paths, bucket policies, SDK defaults, S3 Object Lock, and best-effort support for S3-compatible providers.

Gunnar Morling's excellent post on [leader election with S3 conditional writes](https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/), where each leadership epoch is represented by a new lock object, inspired the storage based lock provider I built for Hudi. The corresponding [Hacker News thread](https://news.ycombinator.com/item?id=41357123) is worth reading too, mostly because distributed-systems purists immediately show up to point out the lease and fencing caveats. They are not wrong.

## The Hudi Lock

The Hudi implementation used one lock file per table:

```text
s3://bucket/table/.hoodie/.locks/table_lock.json
```

The file stored a lock owner, an expiration time, and whether the lock had been explicitly released. Acquisition created or overwrote the file with a conditional write. Renewal extended the expiration with another conditional write. Release marked the file as expired instead of deleting it, because conditional delete was not portable enough for the design.

That design felt clean. Hudi users already needed permission to write table metadata to object storage. If the lock could live next to the table metadata, they did not need a separate coordination system just to make concurrent commits work.

On the happy path, it was easy to explain and cheap to run. A commit needed one read of the lock file, one conditional write to acquire it, renewals while the commit was in progress, and one conditional write to release it. The whole thing was a small number of object-storage interactions around the real table write.

## The Atomic Write Is Not The Whole Protocol

The harder part was everything around that one write. The implementation still had to define what happened when a lease expired, a writer stalled, or an object-store request returned an ambiguous result.

Martin Kleppmann's [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) frames this as the difference between locks used "for efficiency or for correctness." If two workers do the same background job, maybe you waste money. If two writers corrupt a table, you have a correctness bug. In storage systems, correctness almost always comes before performance; performance matters, but only after the protocol can prove it will not corrupt state.

Leases are especially tricky. A process can acquire a lock, pause for longer than the lease (stop-the-world), and then resume after another process has acquired the next lock. The stale process may still believe it is allowed to write. A lock service alone does not fix that. The protected resource needs a way to reject stale work, commonly with fencing tokens or with another conflict-detection mechanism.

We did not add fencing tokens because fencing would have required the Hudi commit path itself to enforce them. The lock provider alone could not make stale writes impossible; it could only coordinate admission and help writers notice when they had lost the lock. In practice, the storage lock fit into Hudi's existing optimistic concurrency flow, where stale or conflicted commits were expected to abandon rather than be accepted blindly.

## The Retry Issue

One subtle issue with this design is that ordinary SDK retries can change the meaning of a conditional-write failure. The failure mode is now described clearly in [AWS SDK Java v2 issue #6580](https://github.com/aws/aws-sdk-java-v2/issues/6580):

1. A client sends `PutObject` with `If-None-Match: *`.
2. The request reaches S3 and creates the object.
3. The response never makes it back to the client, perhaps due to a network issue.
4. The SDK sees a `500` and retries the request.
5. The retry reaches S3, sees the object already exists, and returns `412 Precondition Failed`.

From the protocol's point of view, `412` usually means "someone else won the race." In this case, the client may have won the race itself. Every component behaved reasonably; there is a semantic gap between transport-level retry and protocol-level state.

For a lock service, that ambiguity has to be handled. During acquisition, a retry-induced `412` can look like a lost race. During renewal, a `412` often has to mean "you may no longer own this lock," because assuming otherwise is dangerous.

The Hudi fix was to disable retries in the storage lock clients, captured in [apache/hudi#17869](https://github.com/apache/hudi/pull/17869). A generic SDK retry can repeat the request, but it cannot know whether the original write acquired, renewed, or lost the lock. Keeping retries at the lock-renewal layer made the behavior easier to reason about: a `412` could no longer be caused by the client's own hidden retry.

Once lower-level retries were disabled, transient object-store failures surfaced directly in the lock code. One of those failures was specific to GCS: acquiring and releasing the same lock object in under a second could return `429`, because GCS documents a maximum write rate of [one write per second to the same object name](https://cloud.google.com/storage/quotas). With a single lock file, you can hit that limit even when the table itself is nowhere near an object-store throughput ceiling.

## Recovering From Ambiguous Writes

The retry ambiguity is not unique to Hudi's single-lock-file design. Epochs and fencing tokens help with stale owners, and they can make recovery from an ambiguous write more structured, but they do not make a conditional write magically idempotent.

If a client does not know whether a conditional write succeeded, it has to re-enter the protocol. Reading or listing state is not enough by itself (TOCTOU); the client needs another guarded operation, or a downstream fence, before assuming it owns anything.

Now this "simple" lock protocol has gotten very complex. It needs an explicit ambiguous-write recovery path, separate rules for acquisition and renewal, and some way to avoid read-then-act races. Large-scale systems absorb that complexity with patterns like append-only WAL slots, brokers, group commit, or fencing tokens. Those are useful patterns, but they are also a sign that the one-file lock must become a real distributed protocol.

## Same Primitive, Different Object Stores

The GCS one-write-per-second limit is not just a Hudi anecdote. turbopuffer's post on [building a distributed queue in a single JSON file on object storage](https://turbopuffer.com/blog/object-storage-queue) hits the same shape of problem from another angle: a single `queue.json` file with compare-and-set writes is simple and correct, but every queue operation mutates the same object. Once GCS forces the design toward batching and brokered writes, that can become the contract for Azure and S3 too, unless you want separate protocols per provider.

The same retry question has shown up outside S3 too, but not in a way where you can assume the behavior is identical. In a [Trino issue about intermittent Iceberg commit failures on Azure storage](https://github.com/trinodb/trino/issues/25931#issuecomment-3638416609), a maintainer pointed out that Trino's Azure filesystem also uses `if-none-match: *` for exclusive writes, while the Azure SDK has its own retry layer. That looks similar to the S3 retry problem at the protocol level, but the exact retry behavior, error surface, and configuration knobs belong to the Azure SDK, not the AWS SDK.

It is not enough to ask whether S3, GCS, and Azure all have some version of conditional writes. The important questions are more specific:

- How fast can I mutate the same object?
- Are failed conditional writes billed or throttled differently?
- Are retries automatic, and can I turn them off for this path?
- Does the S3-compatible provider actually implement the exact S3 feature I need?

DigitalOcean Spaces (ceph-based) was a concrete example. It is S3-compatible, but that does not mean every S3 concurrency primitive is there. Its docs list conditional headers for reads like `GetObject` and `HeadObject`, but not the `PutObject` behavior this lock protocol needed. AWS S3 support did not automatically imply Spaces support.

This is a tricky part of multi-cloud support especially with these S3-compatible object stores on the rise. They all do things slightly differently and you have to support a specific set of operations, retry behaviors, limits, and error cases for each provider.

> **Aside: read latency varies a lot by provider too.**
> Simon Eskildsen [shared a benchmark](https://x.com/Sirupsen/status/2050895383866249618) of `GET` with `If-None-Match` across object stores. The operation is useful for cache validation: if a query node has a cached view of the WAL, a `304 Not Modified` means it can keep using that cache without downloading the object again.
>
> The interesting part is how differently the stores performed on the same conditional read. This is a different use case than locks, but same as with my lock service: once object storage is on a hot path, provider behavior is part of the system design.

## My reflections

This was a fascinating and interesting experience; I will certainly build things with conditional writes again. 

One of my biggest takeaways is that there are a lot of nuances beyond the API spec, and those nuances matter when you build critical distributed primitives on top. You have to run the system for a while. Not just a unit test, not just a tight benchmark loop, but something long-lived enough to see retries, throttling, lost responses, slow requests, provider maintenance, and the occasional missing nine. That is when the actual contract starts to show up (or breaks!).

That does not replace careful design. TLA+ is something I wish I had spent more time on. Retries belong at the protocol layer, not hidden inside a generic SDK client. Key layout still matters. A single hot object is easy to reason about, but it is also a bottleneck and may run straight into provider limits.

Yet **the happy path for conditional writes is beautiful:** read a version, write if the version still matches, and let the object store serialize the race.

I am excited to see what people build with conditional requests on object storage over the next few years. Locks, queues, WALs, cache validation, and state files are probably just the obvious first wave. As more systems are designed around object storage as the durable substrate, these small request preconditions will likely become a bigger piece of the conversation around S3 and object storage.