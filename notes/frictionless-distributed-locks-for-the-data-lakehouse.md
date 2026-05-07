---
layout: note
title: "Frictionless Distributed Locks for the Data Lakehouse using Cloud Storage Conditional Write Capabilities"
date: 2026-04-01
---

## Introduction

Coordinating concurrent writes in distributed systems presents a significant engineering challenge. Standard solutions to this problem often involve distributed locks based on a compare-and-swap (CAS) style pattern, which allows one process to say, "only write my new value if no one else has changed it." In the context of data lakehouses, all three popular projects - [Apache Hudi](https://hudi.apache.org/docs/concurrency_control/#distributed-locking) with lock providers, Apache Iceberg with [catalogs](https://iceberg.apache.org/terms/#catalog), and Delta Lake with [log store](https://delta.io/blog/2022-05-18-multi-cluster-writes-to-delta-lake-storage-in-s3/) - depend on locking services of some form to execute [optimistic concurrency control](https://hudi.apache.org/blog/2021/12/16/lakehouse-concurrency-control-are-we-too-optimistic/). Hudi also depends on distributed locks for time generation within its [non-blocking concurrency control](https://hudi.apache.org/blog/2024/12/06/non-blocking-concurrency-control/) mechanisms for streaming writes.  
   
Regardless of whether it's a catalog, a managed service like DynamoDB, or a self-hosted distributed coordination service like Apache Zookeeper, lakehouse users require an additional service to perform multi-cluster, distributed writes to tables. This adds operational complexity and may even become a significant bottleneck for user adoption. In an ideal scenario, writers should be able to safely execute concurrent writes to tables without relying on this external service dependency, solely depending on cloud storage. Cloud storage systems, such as [AWS S3](https://aws.amazon.com/pm/serv-s3), [Azure ADLS](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction), and [GCS,](https://cloud.google.com/storage) provide industry-proven durability and availability guarantees.  
   
Modern object stores, such as Google Cloud Storage (GCS) and Azure Blob Storage, have supported **conditional write** primitives for several years, ensuring atomic operations. However, leveraging cloud storage for distributed coordination has been challenging, particularly with AWS S3, due to the lack of support for CAS-style atomic operations. Fortunately, AWS S3 [recently introduced this support](https://aws.amazon.com/about-aws/whats-new/2024/08/amazon-s3-conditional-writes/). Conditional writes enable direct lock management using the cloud storage system, eliminating external dependencies by leveraging the cloud storage provider's native consistency guarantees.

We'll explore how Apache Hudi implements concurrency control using storage-based locking, powered by cloud storage conditional writes. You'll learn about the core design, how lock acquisition / release / renewal work, and what to consider when configuring this mechanism yourself. In addition, we'll discuss how these behaviors have played out in production, with real-life examples showing the effect that critical design choices have in distributed systems.

## The Challenge of Distributed Locking in Lakehouses

Concurrency control is an essential aspect of a lakehouse architecture that supports multiple workloads on a single table. Typically, this involves one or more ingest/ETL writers bringing new data into the table, with table maintenance services and periodic deletion and backfill jobs in the background. Hence, ensuring safe, concurrent writes in a lakehouse is absolutely critical. Without a robust locking mechanism, two processes could simultaneously modify the same data file, resulting in corrupted data, silent write conflicts, or irreparable table inconsistencies. 

To prevent these scenarios, you need a locking system that can:

* **Ensure exclusive access**: Elect only one leader to perform writes, so no two writers ever try performing a conflicting operation at the same time.  
* **Maintain liveness**: Continuously renew or release the lock as work progresses, ensuring the leader retains ownership while it remains active without requiring manual intervention.  
* **Deadlock avoidance and failure recovery**: Automatically expire locks left behind by crashed or stalled processes, avoiding permanent deadlocks that block all writers.

Apache Hudi supports [multiple concurrency control mechanisms](https://hudi.apache.org/blog/2025/01/28/concurrency-control) to ensure safe and consistent writes in distributed environments. These include **Optimistic Concurrency Control (OCC)**, where writers expect minimal conflict and validate before committing; **Multi-Version Concurrency Control (MVCC)**, which allows concurrent writers and table services to operate on the same snapshot without blocking each other; and **Non-Blocking Concurrency Control (NBCC)**, which allows concurrent writers without livelocking and starvation of OCC.

*Reference: https://hudi.apache.org/blog/2025/01/28/concurrency-control*

For concurrency control, Hudi has historically relied on external coordination services, such as [ZooKeeper,](https://zookeeper.apache.org/doc/r3.1.2/zookeeperOver.html) [DynamoDB,](https://aws.amazon.com/blogs/database/building-distributed-locks-with-the-dynamodb-lock-client/) or [Hive Metastore](https://hive.apache.org/), which adds an operational burden to users. Users or infrastructure teams inside companies must deploy, secure, and monitor these services, and each lock request incurs extra network hops and potential latency. Under heavy cluster load or in geographically distributed deployments, these factors can introduce unpredictable delays and even single points of failure. Leveraging built-in atomic operations in your cloud storage layer offers a simpler, more resilient alternative.

## Leveraging Conditional Writes in Cloud Storage

Cloud object stores now support precondition headers on write operations:

[**AWS S3**](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html)

* An "If-None-Match: \*" precondition succeeds only if the object does *not* yet exist.  
* An "If-Match" precondition succeeds only if the object's current ETag (a unique hash of its payload) matches a specified value.

[**Google Cloud Storage**](https://cloud.google.com/storage/docs/request-preconditions)

* An "if_generation_match=0" parameter to succeed only if no live version exists.  
* A generation-based match to update an object only if its current generation matches a given number.

[**Azure Blob Storage**](https://learn.microsoft.com/en-us/rest/api/storageservices/specifying-conditional-headers-for-blob-service-operations)

* Uses the same preconditions as S3

These primitives provide the foundational blocks for implementing a compare-and-swap operation, with atomic guarantees necessary for distributed locking. However, we need to build a robust locking protocol for leader election, renewal, and safe release, implemented as an easy-to-use library to leverage these new primitives for lakehouse concurrency control. 

## Design Overview

In this section, we outline a design for a new distributed locking library for Apache Hudi that can also be applied to other projects. The new [StorageBasedLockProvider](https://github.com/apache/hudi/blob/master/rfc/rfc-91/rfc-91.md) in Hudi 1.1 can be easily enabled for concurrency control by AWS and Google Cloud users. Each table in Hudi that uses this lock provider maintains a single lock file, **table_lock.json**, inside a **.hoodie/.locks** folder in its metadata directory that all concurrent writers can access. 

This file contains three key fields:

* **owner**: a unique identifier (UUID) for the lock holder.  
* **expiration**: a timestamp indicating when the lock will expire.  
* **expired**: a flag denoting whether the lock has been explicitly released.

Using this [lock provider in hudi](https://hudi.apache.org/docs/next/concurrency_control/#storage-with-conditional-writes-based) is straightforward; simply add the following to your table's configuration.

```shell
hoodie.write.lock.provider = org.apache.hudi.client.transaction.lock.StorageBasedLockProvider 
```

Since the lock file is just a file in the same directory as the existing table, all necessary permissions to write to this lock file will already be validated. Under the hood, each storage-specific host, such as s3://, s3a://, gs://, is mapped to an implementation of StorageLockClient, which defines a contract for using conditional requests to write to different cloud storage services. Currently, only S3 and GCS are supported, but each new storage service implementation only needs to implement this interface.

Three operations drive all coordination:

1. **Lock Acquisition**  
2. **Lock Renewal**  
3. **Lock Release**

A lightweight **heartbeat manager** periodically extends the expiration by renewing the lock to keep the ownership intact. If a process crashes or becomes unresponsive, its lock naturally expires, allowing other contenders to proceed. Let's understand these 3 operations in detail:

### 1. Lock Acquisition

There are few different scenarios to handle, for a process attempting to acquire the lock. If the lock is acquired, the expiration is set to a default of 5 minutes from the current epoch. 

* **No existing lock file**: attempt to create the file using a conditional write that only succeeds if the file does not yet exist. If the write succeeds, this process becomes the lock owner and starts the heartbeat.  
* **Expired lock and conditional write succeeds**: read the current metadata (ETag or generation) of the lock file. Then, attempt an overwrite using a conditional write that only succeeds if the stored metadata matches the value you observed. A successful overwrite grants ownership.  
* **Expired lock and conditional write fails**: performs the same actions as above, but another process simultaneously writes to the lock file and succeeds first. This changes the existing ETag, and which causes the second writer to encounter precondition failure milliseconds later.  
* **Existing and still valid lock**: the lock acquisition exits and does not attempt to write a file, indicating another process currently holds the lock. The new process does not proceed.

This approach ensures that exactly one process can become the leader and acquire a lock, based solely on storage-layer atomicity, even if there are multiple concurrent writers trying to start transactions.

### 2. Lock Renewal

Without a mechanism to periodically renew the lock, the failure of a process that holds the lock will lead to a deadlock (no other process can acquire the lock). Thus, each lock must have an expiry and each process that has acquired a lock is required to renew it continuously using the heartbeat manager.

* The heartbeat manager runs a thread in the background at a configurable interval (by default every 30 seconds). On each tick, it issues a conditional update to extend the expiration (by a default of 5 minutes from the current epoch). It will continue extending the expiration until it is shut down as a part of the unlocking mechanism. *Design note: it does not need to read the state of the lock file. If another process has modified the lockfile, the error code returned by the conditional write will inform the original writer.*   
* The conditional update uses the same logic as the lock acquisition. The ETag of the most recently written lock file will be used to ensure the heartbeat manager remains the owner of the lock. Even if the update fails, the update will keep retrying every 30 seconds.

This heartbeat ensures that as long as a process is healthy and running, its lock remains valid.

### 3. Lock Release

In the happy scenario, the process holding a lock is able to gracefully signal that it has now released the lock, for others to use.

* When work is complete, the owner marks the lock as expired via a conditional update that verifies the stored metadata.  
* No physical deletion of the lock file is necessary. Setting the expired flag is sufficient to allow new contenders to acquire the lock.

By avoiding deletion, we eliminate race conditions where two contenders might simultaneously check for non-existence and both succeed. By marking the lock as expired, we ensure any processing waiting on the lock can immediately attempt to acquire it.

## Why this design is correct and efficient

### Correctness

* **Mutual Exclusion:** Acquiring the lock is one atomic conditional-write. The cloud store (via ETag/version) ensures that only one client can succeed when the file doesn't exist or is expired. Two processes cannot hold the lock simultaneously.  
* **No Stale Owners:** Renewal also uses a conditional update against the current version. If the TTL already expired (or someone else grabbed it), the renewal fails, so a process won't mistakenly think it still owns the lock.  
* **Automatic Failure Recovery:** If a holder crashes or loses network connectivity, it simply stops sending heartbeats. Once the TTL passes, the lock disappears, and another contender can take over.

### Efficiency

* **No Extra Infrastructure:** ZooKeeper/etcd or DynamoDb is not needed. The cloud object store itself enforces atomic check+update through its existing ETag/version mechanism.  
* **Low amount of I/O and Metadata:** Only three calls are required at minimum per lock lifecycle:  
  1. One call to get the lock status (read).  
  2. One conditional write to create/claim (acquire) with periodic conditional writes to extend expiration (heartbeat).  
  3. One conditional write to release (unlock). There's no polling loop, no extra tables, and no watch channels, just the lock file itself.  
* **Tunable Performance vs. Safety:** You choose a TTL (for example, a few seconds to accommodate GC pauses). As long as you renew before that expiration, the lock stays valid. A short TTL bounds how long someone waits after a crash; a longer TTL tolerates more transient network hiccups.  
* **Resilient to Transient Store Glitches:** If the object store is briefly unavailable when you send a heartbeat, retries kick in automatically. A momentary failure won't immediately revoke the lock.

## Handling Edge Cases

In distributed systems, edge cases aren't just theoretical, they're the norm. Failures from network partitions, process crashes, and clock drift occur frequently enough that **robust handling of these scenarios is critical for correctness**. A lock mechanism that works 99% of the time but fails silently under rare conditions can cause severe data corruption or outages. This storage based lock provider adds several defenses against those edge cases to maintain safe lock ownership and renewal behavior.

**Process Failures**

* The heartbeat manager captures the original lock owner thread upon starting renewal. If this thread dies, the heartbeat manager will stop renewal. If the renewal process fails, the heartbeat manager will interrupt the lock owner thread.  
* If the lock fails to be released, an error will be thrown and the transaction will be aborted. This is a last resort defense against concurrent writers.  
* Shutdown hooks are added to ensure that failed writers do not leave dangling locks which might force competing writers to wait a few minutes before acquiring the lock themselves.

**Clock Drift**

* A small clock-skew tolerance (for example, 500 ms) is assumed. This means each comparison between a node's current time and the current lock file's expiration time in cloud storage is reduced by 500ms. All writers should be in the same cloud to ensure they rely on that cloud provider's NTP server. Yugabyte has a [great blog](https://www.yugabyte.com/blog/aws-clock-synchronization/) where they discuss their maximum allowable drift for AWS. In practice, it should not be more than 50ms within the same cloud. For Hudi, locks are acquired during time generation and action state changes, using [TrueTime](https://hudi.apache.org/docs/timeline/#truetime-generation) principles.

Distributed locking in general [does not provide absolute guarantees](https://news.ycombinator.com/item?id=41357123) with multi writer contention. Any system with multiple nodes will have very rare cases of failure points, but if you can reduce the occurrence such that it is negligible (e.g. eleven 9s), then these systems can be used in production.

## Storage based distributed locking in production

**2025 Outages**

During the June GCP outage, we saw the renewal retry logic on full display in production. All production tables using the GCS implementation of the StorageBasedLockProvider were failing with 5XXs on one quarter of requests to GCS, including the requests to renew locks every 30 seconds. However, every lock's heartbeat was eventually able to self-heal and complete the transaction over the course of the 1 hr outage.

Then, during the East US 1 outage with AWS in October, our customers who relied on DynamoDb for locking saw errors throughout the day, and their pipelines were stuck, while customers using the storage based lock provider were significantly less impacted by the outage.

### The Peril of Retries: A Critical Discovery in Conditional Writes

During our implementation of a storage-based lock provider, we encountered a significant challenge related to **idempotency and retries**. S3's conditional writes operate with exactly-once write semantics. Consequently, any client-side retry (e.g., from an SDK) against a conditional write can unexpectedly result in a 412 Precondition Failed error.

The danger lies in how a client interprets this 412 error.

* **During Lock Renewal:** The client must assume the worst: that the lock has been acquired by another process, and it must immediately interrupt the writing thread.  
* **During Initial Lock Acquisition:** A retry-induced 412 can create "ghost locks." The lock acquirer mistakenly believes another entity holds the lock, when in reality, the 412 was a byproduct of its own retry.

This behavior highlights a fundamental principle of [distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html): reading remote state (who owns the lock) and then acting on it (starting the heartbeat/transaction) is an immediate safety violation. The system is vulnerable because any subsequent event, such as a GC pause, could cause the lock to be lost before the transaction begins.

This specific flakiness was uncovered during a week of S3 instability, where **five separate conditional writes** out of millions, across multiple regions and data planes, returned a 412 due to a retry. While the transactions guarded by these locks failed gracefully as expected, the issue generated confusing logs and alerts because the lock provider had no way to distinguish a retry failure from a genuine concurrent writer.

Retries have since been disabled. The discovery underscored the vital lesson that when building critical distributed systems on top of external services, one must comprehensively understand every possible way the remote calls can be returned and interpreted.

## Conclusion

By harnessing **conditional-write** capabilities of modern cloud object stores, you can build a **fully native, storage-based locking** mechanism that:

* **Eliminates external lock services**, reducing operational complexity.  
* **Simplifies deployment** and maintenance by leveraging existing storage infrastructure.  
* **Ensures robust optimistic concurrency** with built-in atomicity and expiration semantics.

This pattern provides a highly available and maintainable solution for distributed locking in your lakehouse, allowing you to focus on data processing rather than managing additional coordination layers.

Embracing storage based locking further extends Hudi's design principles as being an open storage system that only depends on storage at runtime for its transactional capabilities. Specifically, it allows Hudi to support different catalogs, without being siloed into a single one and forcing users to deploy a catalog to be able to use Hudi.

