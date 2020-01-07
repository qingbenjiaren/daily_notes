# WiredTiger Storage Engine

Starting in MongoDB 3.2, the WiredTiger storage engine is the default storage engine. For existing deployments, if you do not specify the `--storageEngine` or the [`storage.engine`](https://docs.mongodb.com/manual/reference/configuration-options/#storage.engine) setting, the version 3.2+ [`mongod`](https://docs.mongodb.com/manual/reference/program/mongod/#bin.mongod) instance can automatically determine the storage engine used to create the data files in the `--dbpath` or [`storage.dbPath`](https://docs.mongodb.com/manual/reference/configuration-options/#storage.dbPath). See [Default Storage Engine Change](https://docs.mongodb.com/manual/release-notes/3.2-compatibility/#storage-engine-compatibility).

**Document Level Concurrency**

WiredTiger uses *document-level* concurrency control for write operations. As a result, multiple clients can modify different documents of a collection at the same time.

For most read and write operations, WiredTiger uses optimistic concurrency control. WiredTiger uses only intent locks at the global, database and collection levels. When the storage engine detects conflicts between two operations, one will incur a write conflict causing MongoDB to transparently retry that operation.

Some global operations, typically short lived operations involving multiple databases, still require a global “instance-wide” lock. Some other operations, such as dropping a collection, still require an exclusive database lock.

**Snapshots and Checkpoints**

WiredTiger uses MultiVersion Concurrency Control (MVCC). At the start of an operation, WiredTiger provides a point-in-time snapshot of the data to the operation. A snapshot presents a consistent view of the in-memory data.

When writing to disk, WiredTiger writes all the data in a snapshot to disk in a consistent way across all data files. The now-[durable](https://docs.mongodb.com/manual/reference/glossary/#term-durable) data act as a *checkpoint* in the data files. The *checkpoint* ensures that the data files are consistent up to and including the last checkpoint; i.e. checkpoints can act as recovery points.

Starting in version 3.6, MongoDB configures WiredTiger to create checkpoints (i.e. write the snapshot data to disk) at intervals of 60 seconds. In earlier versions, MongoDB sets checkpoints to occur in WiredTiger on user data at an interval of 60 seconds or when 2 GB of journal data has been written, whichever occurs first.

During the write of a new checkpoint, the previous checkpoint is still valid. As such, even if MongoDB terminates or encounters an error while writing a new checkpoint, upon restart, MongoDB can recover from the last valid checkpoint.

​		--个人理解：若设置为60秒创建一个检查点，在不适用日志的前提下，mongo可恢复至60s前创建的checkpoint

The new checkpoint becomes accessible and permanent when WiredTiger’s metadata table is atomically updated to reference the new checkpoint. Once the new checkpoint is accessible, WiredTiger frees pages from the old checkpoints.

Using WiredTiger, even without [journaling](https://docs.mongodb.com/manual/core/wiredtiger/#storage-wiredtiger-journal), MongoDB can recover from the last checkpoint; however, to recover changes made after the last checkpoint, run with [journaling](https://docs.mongodb.com/manual/core/wiredtiger/#storage-wiredtiger-journal).

**Journal**

WiredTiger uses a **write-ahead log (i.e. journal)** in **combination** with **[checkpoints](https://docs.mongodb.com/manual/core/wiredtiger/#storage-wiredtiger-checkpoints)** to ensure data durability.

**The WiredTiger journal persists all data modifications between checkpoints**. If MongoDB exits between checkpoints, it **uses the journal to replay all data modified since the last checkpoint.** For information on the frequency with which MongoDB writes the journal data to disk, see [Journaling Process](https://docs.mongodb.com/manual/core/journaling/#journal-process).