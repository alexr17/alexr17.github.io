---
layout: note
title: "What I Learned about Building Distributed Locks on S3 and GCS"
date: 2026-04-01
---

When I was working at Onehouse, I implemented a distributed locking service for Apache Hudi's optimistic concurrency control using conditional writes in object storage.

The idea was simple enough to fit in one sentence: instead of requiring a separate lock service like ZooKeeper or DynamoDB, use the atomic write preconditions already available in cloud object stores. I wrote the original Hudi design up in [RFC-91: Storage-based lock provider using conditional writes](https://github.com/apache/hudi/blob/master/rfc/rfc-91/rfc-91.md). The RFC is the dry version. This is the version about why the primitive is so useful, and why it is much easier to get wrong than it looks.

Conditional writes are a great distributed systems primitive. They are not a lock service by themselves.

That distinction matters. Once retries, leases, cache freshness, provider throttling, and S3-compatible-but-not-quite-S3 APIs enter the picture, the hard part is not "does my object store support this header?" The hard part is understanding the exact protocol contract you are building on top of that header. For experienced distributed-systems engineers, this may all look familiar. For me, it was new terrain. I started this project two months into Onehouse, after mostly building backend APIs for MySQL at Microsoft, with barely any S3 experience.

## The Tiny API That Changes The Design Space

The core primitive is compare-and-set for object storage.

For a long time, S3 was the awkward missing piece here. GCS and Azure had supported object preconditions for years, but S3 did not support the create-if-absent write path that many of these designs want. That changed in August 2024, when [S3 added conditional writes for `PutObject` and `CompleteMultipartUpload`](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes/).

On S3, `If-None-Match: *` says "create this object only if it does not already exist." `If-Match` says "overwrite this object only if its current ETag is the one I observed." GCS exposes the same idea through generation preconditions, including `if_generation_match=0` for create-if-absent. Azure Blob Storage has its own version of conditional request headers.

That is enough to build useful coordination protocols:

- only one writer creates a file for a given logical slot
- only one process updates a lock file from the version it observed
- a reader can cheaply ask whether cached data is still current

You can see the same pattern showing up elsewhere. turbopuffer has talked publicly about S3 conditional writes being important to its architecture, and their [first-principles object-storage database discussion](https://turbopuffer.com/blog/podcast-database-from-first-principles) gets at the same underlying problem: when multiple writers append to a WAL on object storage, how do you make sure exactly one writer claims the next durable slot? Terraform is another good example. In Terraform 1.10, the S3 backend added optional native state locking with a lock file written using `If-None-Match`, removing the need for DynamoDB in that path. Bruno Schaatsbergen's writeup on [S3 native state locking](https://www.bschaatsbergen.com/s3-native-state-locking) shows the same tradeoff from a very different ecosystem: the primitive removes infrastructure, but the implementation still has to care about migration paths, bucket policies, SDK defaults, S3 Object Lock, and best-effort support for S3-compatible providers.

Gunnar Morling's excellent post on [leader election with S3 conditional writes](https://www.morling.dev/blog/leader-election-with-s3-conditional-writes/), where each leadership epoch is represented by a new lock object, inspired the storage based lock provider I built for Hudi. The corresponding [Hacker News thread](https://news.ycombinator.com/item?id=41357123) is worth reading too, mostly because distributed-systems purists immediately show up to point out the lease and fencing caveats. They are not wrong.

Object storage has stopped being "a place to put blobs" and started becoming a place where you can create small, durable ordering facts.

## The Hudi Lock

The Hudi implementation used one lock file per table:

```text
s3://bucket/table/.hoodie/.locks/table_lock.json
```

The file stored a lock owner, an expiration time, and whether the lock had been explicitly released. Acquisition created or overwrote the file with a conditional write. Renewal extended the expiration with another conditional write. Release marked the file as expired instead of deleting it, because conditional delete was not portable enough for the design.

That design removed a real operational dependency. Hudi users already needed permission to write table metadata to object storage. If the lock could live next to the table metadata, they did not need a separate coordination system just to make concurrent commits work.

But none of this made object storage a consensus system. It gave us one atomic operation. The rest of the protocol still had to be engineered.

## The Atomic Write Is Not The Whole Protocol

Martin Kleppmann's [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) is still the right thing to keep in your head when building anything like this. First ask whether the lock protects efficiency or correctness. If two workers do the same background job, maybe you waste money. If two writers corrupt a table, you have a correctness bug.

In storage systems, this is almost always in the correctness bucket. Performance matters, but only after the protocol can prove it will not corrupt state. For Hudi, a slower commit path would have been annoying. A fast commit path that occasionally let two writers believe they both owned the table would have been unusable.

Leases are especially tricky. A process can acquire a lock, pause for longer than the lease (stop-the-world), and then resume after another process has acquired the next lock. The stale process may still believe it is allowed to write. A lock service alone does not fix that. The protected resource needs a way to reject stale work, commonly with fencing tokens or with another conflict-detection mechanism.

In other words, the conditional write was just one piece of the Hudi concurrency story.

## The Retry Issue

One subtle issue with this design is that ordinary SDK retries can change the meaning of a conditional-write failure. The failure mode is now described clearly in [AWS SDK Java v2 issue #6580](https://github.com/aws/aws-sdk-java-v2/issues/6580):

1. A client sends `PutObject` with `If-None-Match: *`.
2. The request reaches S3 and creates the object.
3. The response never makes it back to the client.
4. The SDK retries the request.
5. The retry reaches S3, sees the object already exists, and returns `412 Precondition Failed`.

It is also worth noting that when this happens, the retry is often just the visible symptom. The underlying issue may be a network blip, a lost response, or S3 briefly failing to live up to one of its many nines.

From the protocol's point of view, `412` usually means "someone else won the race." In this case, the client may have won the race itself. Every component behaved reasonably; there is a semantic gap between transport-level retry and protocol-level state.

For a lock service, that ambiguity has to be handled. During acquisition, a retry-induced `412` can look like a lost race. During renewal, a `412` often has to mean "you may no longer own this lock," because assuming otherwise is dangerous.

The Hudi fix was to disable retries in the storage lock clients, captured in [apache/hudi#17869](https://github.com/apache/hudi/pull/17869). A generic SDK retry can repeat the request, but it cannot know whether the original write acquired, renewed, or lost the lock, so Hudi kept retries at the lock-renewal layer instead. That was the right direction, but it was not free.

Turning off lower-level retries meant that more transient object-store failures reached the lock code paths directly. It also made provider limits harder to ignore. GCS documents a maximum write rate of [one write per second to the same object name](https://cloud.google.com/storage/quotas), which matters a lot for single-object designs. A lock file is intentionally a hot object. That is the whole point. But object stores are usually optimized for high aggregate throughput across keys, not rapid mutation of one key.

## Recovering From Ambiguous Writes

The retry ambiguity is not unique to Hudi's single-lock-file design. Epochs and fencing tokens help with stale owners, and they can make recovery from an ambiguous write more structured, but they do not make a conditional write magically idempotent.

If a client does not know whether a conditional write succeeded, it has to re-enter the protocol. Reading or listing state is not enough by itself (TOCTOU); the client needs another guarded operation, or a downstream fence, before assuming it owns anything.

Now this "simple" lock protocol has gotten very complex. It needs an explicit ambiguous-write recovery path, separate rules for acquisition and renewal, and some way to avoid read-then-act races. Large-scale systems absorb that complexity with patterns like append-only WAL slots, brokers, group commit, or fencing tokens. Those are useful patterns, but they are also a sign that the one-file lock must become a real distributed protocol.

## Shared protocol for multi-cloud

The GCS one-write-per-second limit is not just a Hudi anecdote. turbopuffer's post on [building a distributed queue in a single JSON file on object storage](https://turbopuffer.com/blog/object-storage-queue) hits the same shape of problem from another angle.

Their simplest queue is one `queue.json` file. Pushers and workers read the file, modify it, and write it back with compare-and-set. It is a clean design: one durable file, strong consistency, no separate coordination service. It also has an obvious ceiling. If every queue operation mutates the same object, the system is bounded by object rewrite latency and provider limits. On GCS, the single-object write limit forces the design toward batching and brokered writes; they likely use the same protocol for Azure and S3 as well.

It is not enough to ask whether S3, GCS, and Azure all have some version of conditional writes. The important questions are more specific:

- How fast can I mutate the same object?
- Are failed conditional writes billed or throttled differently?
- Are retries automatic, and can I turn them off for this path?
- Is the condition checked on `PutObject`, multipart completion, copy, metadata update, or only reads?
- Does the S3-compatible provider actually implement the exact S3 feature I need?

That last point came up with DigitalOcean Spaces. Spaces is S3-compatible, but "S3-compatible" does not mean "supports every S3 concurrency primitive." In the compatibility docs, conditional headers are listed for reads like `GetObject` and `HeadObject`, while the `PutObject` header list does not expose the create-if-absent/write-if-version behavior this kind of lock protocol needs. A design that works on AWS S3 with the AWS SDK may not work on every S3-compatible object store, and the only real answer is to test the exact operation against each provider.

This is a tricky part of multi-cloud support with all these S3-compatible object stores on the rise. They all do things slightly differently and you have to support a specific set of operations, retry behaviors, limits, and error cases for each provider.

## Conditional Reads Matter Too

There is a read-side version of this story.

Simon Eskildsen from turbopuffer [pointed out](https://x.com/Sirupsen/status/2050895383866249618) that `GET` with `If-None-Match` is important for building fast databases on object storage. If a query node has a cached view of the WAL, it can ask the object store whether the relevant object has changed. A `304 Not Modified` means the cache is still fresh without downloading the object again.

That is a small feature with large architectural consequences. Conditional writes help establish ordering or ownership facts. Conditional reads help validate cached state cheaply. In an object-storage database, both can sit on hot paths.

They also make provider differences extremely visible. Simon shared a benchmark of `GET 304` latency across providers where Azure looked surprisingly strong among the big three. The exact numbers are less important than the shape of the lesson: once object storage is in the read path of a database, tail latency of conditional reads becomes a database concern, not a storage trivia question.

## What I learned

I will build things with conditional writes again. The key is treating them as part of the protocol, not as a storage API trick.

The biggest thing I learned is that object storage behavior is hard to understand from API docs and small benchmarks alone. You have to run the system for a while. Not just a unit test, not just a tight benchmark loop, but something long-lived enough to see retries, throttling, lost responses, slow requests, provider maintenance, and the occasional missing nine. That is when the actual contract starts to show up.

The public Trino issue trail is a good example. Trino added S3 exclusive create, and the hardening came later: [trinodb/trino#27400](https://github.com/trinodb/trino/issues/27400) documented how a write could succeed on S3, lose the response, get retried by the AWS SDK, and then surface as `FileAlreadyExistsException`. That is the kind of behavior you find by running real code against real object storage long enough for the weird cases to happen.

That does not replace careful design. Retries still belong at the protocol layer, not hidden inside a generic SDK client. Key layout still matters. A single hot object is easy to reason about, but it is also a bottleneck and may run straight into provider limits. Compatibility testing has to cover the behavioral contract, not just the happy-path API call: retries, error codes, multipart uploads, throttling, and conditional reads.

The happy path for conditional writes is beautiful: read a version, write if the version still matches, and let the object store serialize the race. That is a powerful tool. The unhappy paths are where the system is actually designed.

Object storage has a few extremely useful primitives, and they can replace a surprising amount of infrastructure when the workload fits. Those primitives are sharp. The way to get comfortable with the sharp edges is to design for them, then put the design under sustained load and see what the object store actually does.

I am excited to see what people build with conditional requests on object storage over the next few years. Locks, queues, WALs, cache validation, and state files are probably just the obvious first wave. As more systems are designed around object storage as the durable substrate, these small request preconditions will likely show up in places that currently depend on a separate coordination layer.
