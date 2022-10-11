# Concurrency Control

Concurrency Control is a mechanism that maintains atomicity and isolation, which are two properties of the ACID, when several transactions run concurrently in the database.

## Transaction Isolation Level in PostgreSQL

| Isolation Level  | Dirty Reads            | Non-repeatable Read | Phantom Read           | Serialization Anomaly |
| ---------------- | ---------------------- | ------------------- | ---------------------- | --------------------- |
| READ UNCOMMITTED | Allowed, but not in PG | Possible            | Possible               | Possible              |
| READ COMMITTED   | Not possible           | Possible            | Possible               | Possible              |
| REPEATABLE READ  | Not possible           | Not possible        | Allowed, but not in PG | Possible              |
| SERIALIZABLE     | Not possible           | Not possible        | Not possible           | Not possible          |

In PostgreSQL, you can request any of the four standard transaction isolation levels, but internally only three distinct isolation levels are implemented, i.e., PostgreSQL's `Read Uncommitted` mode behaves like `Read Committed`.

## Transaction ID

Whenever a transaction begins, a unique identifier, referred to as a **transaction id (txid)**, is assigned by the transaction manager. PostgreSQL's txid is a 32-bit unsigned integer, approximately 4.2 billion (thousand millions).

PostgreSQL reserves the following three special txids:

- 0 means Invalid txid.
- 1 means Bootstrap txid, which is only used in the initialization of the database cluster.
- 2 means Frozen txid

Since the txid space is insufficient in practical systems, PostgreSQL treats the txid space as a circle. The previous 2.1 billion txids are `in the past`, and the next 2.1 billion txids are `in the future`.

![](https://user-images.githubusercontent.com/17776979/194903908-28157616-a6bf-49c7-98f2-28f5b6b18555.png)

**Transaction wraparound problem**

Assume that tuple `Tuple_1` is inserted with a **txid** of `100`, i.e. the **t_xmin** of `Tuple_1` is `100`. The server has been running for a very long period and `Tuple_1` has not been modified.

The current **txid** is 2.1 billion + 100 and a SELECT command is executed. At this time, `Tuple_1` is visible because **txid** `100` is in the past. Then, the same SELECT command is executed; thus, the current **txid** is 2.1 billion + 101. However, `Tuple_1` is no longer visible because **txid** `100` is in the future. This is the so called transaction wraparound problem in PostgreSQL.

![](https://user-images.githubusercontent.com/17776979/194998664-b3771956-e82d-4740-ae75-8a19252e913a.png)

To deal with this problem, PostgreSQL introduced a concept called frozen txid, and implemented a process called FREEZE.

In PostgreSQL, a frozen txid, which is a special reserved **txid** `2`, is defined such that it is always older than all other txids. In other words, the frozen txid is always inactive and visible.

The freeze process is invoked by the vacuum process. The freeze process scans all table files and rewrites the **t_xmin** of tuples to the **frozen txid** if the **t_xmin** value is older than the current **txid** minus the [vacuum_freeze_min_age](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-VACUUM-FREEZE-MIN-AGE) (the default is 50 million).


## Tuple Structure

A heap tuple comprises three parts, i.e. the HeapTupleHeaderData structure, NULL bitmap, and user data.

![](https://user-images.githubusercontent.com/17776979/194904177-3c57b02e-9c14-45bb-a25d-a7283e20ad4a.png)

While the HeapTupleHeaderData structure contains seven fields, four fields are required in the subsequent sections.

- **t_xmin** holds the txid of the transaction that inserted this tuple.
- **t_xmax** holds the txid of the transaction that deleted or updated this tuple. If this tuple has not been **deleted** or **updated**, t_xmax is set to 0, which means INVALID.
- **t_cid** holds the command id (cid), which means how many SQL commands were executed before this command was executed within the current transaction beginning from 0. For example, assume that we execute three INSERT commands within a single transaction: `BEGIN; INSERT; INSERT; INSERT; COMMIT;`. If the first command inserts this tuple, t_cid is set to 0. If the second command inserts this, t_cid is set to 1, and so on.
- **t_ctid** holds the tuple identifier (tid) that points to itself or a new tuple tid, is used to identify a tuple within a table. When this tuple is updated, the t_ctid of this tuple points to the new tuple; otherwise, the t_ctid points to itself.

The image below shows an example of how tuples are represented.

![](https://user-images.githubusercontent.com/17776979/194905084-1cb8aafd-3a25-44a8-8833-3b6a582be4a4.png)

## Inserting, Deleting and Updating Tuples

### Insertion

![](https://user-images.githubusercontent.com/17776979/194905163-33e57f5b-8fe8-402f-8889-8adc8f761917.png)

Suppose that a tuple is inserted in a page by a transaction whose **txid** is `99`. In this case, the header fields of the inserted tuple are set as follows.

- **t_xmin** is set to `99` because this tuple is inserted by txid 99.
- **t_xmax** is set to `0` because this tuple has not been deleted or updated.
- **t_cid** is set to `0` because this tuple is the first tuple inserted by txid 99.
- **t_ctid** is set to `(0,1)`, which points to itself, because this is the latest tuple.

### Deletion

![](https://user-images.githubusercontent.com/17776979/194905188-848b481f-fa22-454d-8745-b85399f36e88.png)

Suppose that Tuple_1 is deleted by txid 111. In this case, the header fields of Tuple_1 are set as follows.

- **t_xmax** is set to 111.

Dead tuples should eventually be removed from pages. Cleaning dead tuples is referred to as **VACUUM** processing,

### Update

![](https://user-images.githubusercontent.com/17776979/194905206-f760c912-305a-4e86-9dee-52f130462214.png)

Suppose that the row, which has been inserted by **txid** `99`, is updated twice by **txid** `100`.

When the first `UPDATE` command is executed, `Tuple_1` is logically deleted by setting **txid** `100` to the **t_xmax**, and then `Tuple_2` is inserted. Then, the **t_ctid** of `Tuple_1` is rewritten to point to `Tuple_2`. The header fields of both `Tuple_1` and `Tuple_2` are as follows.

Tuple_1:

- **t_xmax** is set to 100.
- **t_ctid** is rewritten from (0, 1) to (0, 2).

Tuple_2:

- **t_xmin** is set to 100.
- **t_xmax** is set to 0.
- **t_cid** is set to 0.
- **t_ctid** is set to (0,2).

When the second `UPDATE` command is executed, as in the first `UPDATE` command, `Tuple_2` is logically deleted and `Tuple_3` is inserted. The header fields of both `Tuple_2` and `Tuple_3` are as follows.

Tuple_2:

- **t_xmax** is set to 100.
- **t_ctid** is rewritten from (0, 2) to (0, 3).

Tuple_3:

**t_xmin** is set to 100.
**t_xmax** is set to 0.
**t_cid** is set to 1.
**t_ctid** is set to (0,3).

As with the delete operation, if **txid** 100 is committed, `Tuple_1` and `Tuple_2` will be dead tuples, and, if **txid** 100 is aborted, `Tuple_2` and `Tuple_3` will be dead tuples.

## Commit Log (clog)

PostgreSQL holds the statuses of transactions in the Commit Log. The Commit Log, often called the clog, is allocated to the shared memory, and is used throughout transaction processing.

### Transaction Status

PostgreSQL defines four transaction statuses

- IN_PROGRESS
- COMMITTED
- ABORTED
- SUB_COMMITTED (for sub-transaction)

### How Clog Performs

The clog comprises one or more 8 KB pages in shared memory. The clog logically forms an array. The indices of the array correspond to the respective transaction ids, and each item in the array holds the status of the corresponding transaction id.

![](https://user-images.githubusercontent.com/17776979/194907676-68b13396-f29a-4ee8-8368-a79d11e1eeff.png)

### Maintenance of the Clog

When PostgreSQL shuts down or whenever the checkpoint process runs, the data of the clog are written into files stored under the **pg_xact** subdirectory. These files are named 0000, 0001, etc. The maximum file size is 256 KB.

When PostgreSQL starts up, the data stored in the pg_xact's files are loaded to initialize the clog.

The size of the clog continuously increases because a new page is appended whenever the clog is filled up. However, not all data in the clog are necessary. Vacuum processing, regularly removes such old data (both the clog pages and files).

## Transaction Snapshot

A **transaction snapshot** is a dataset that stored information about whether all transactions are active, at a certain point in time for an individual transaction.

The textual representation of the txid_current_snapshot is `xmin:xmax:xip_list`, and the components are described as follows.

- **xmin** earliest txid that is still active. All earlier transactions will either be committed and visible, or rolled back and dead.
- **xmax** first as-yet-unassigned txid. All txids greater than or equal to this are not yet started as of the time of the snapshot, and thus invisible.
- **xip_list** the list includes only active txids between xmin and xmax

For example, in the snapshot `100:104:100,102`, xmin is `100`, xmax `104`, and xip_list `100,102`.

![](https://user-images.githubusercontent.com/17776979/194908644-c598085e-5edd-40c1-af7f-aaa19923ecd4.png)

Transaction snapshots are provided by the **transaction manager**.

- In the READ COMMITTED isolation level, the transaction obtains a snapshot whenever an SQL command is executed
- In REPEATABLE READ or SERIALIZABLE, the transaction only gets a snapshot when the first SQL command is executed.

![](https://user-images.githubusercontent.com/17776979/194909635-d5278bef-c9bf-4a80-ab27-88f135fe6f9d.png)

## Preventing Lost Updates

A **Lost Update**, also known as a **ww-conflict**, is an anomaly that occurs when concurrent transactions update the same rows, and it must be prevented in both the REPEATABLE READ and SERIALIZABLE levels.

![](https://user-images.githubusercontent.com/17776979/194910105-f2b0e8c8-543d-4f37-a2ab-5dfda190a1d6.png)

**The target row is being updated**

`Being updated` means that the row is updated by another concurrent transaction and its transaction has not terminated. In this case, the current transaction must wait for termination of the transaction that updated the target row because PostgreSQL's SI uses the **first-updater-win** scheme. For example, assume that transactions Tx_A and Tx_B run concurrently, and Tx_B attempts to update a row; however, Tx_A has updated it and is still in progress. In this case, Tx_B waits for the termination of Tx_A.

After the transaction that updated the target row commits, the update operation of the current transaction proceeds. If the current transaction is in the READ COMMITTED level, the target row will be updated; otherwise (REPEATABLE READ or SERIALIZABLE), the current transaction is aborted immediately to prevent lost updates.

**The target row has been updated by the concurrent transaction**

The current transaction attempts to update the target tuple; however, the other concurrent transaction has updated the target row and has already been committed. In this case, if the current transaction is in the READ COMMITTED level, the target row will be updated; otherwise, the current transaction is aborted immediately to prevent lost updates.

**There is no conflict**

When there is no conflict, the current transaction can update the target row.

## Serializable Snapshot Isolation

### Write skew problem

Write skew is the problem when two concurrent transactions each determine what they are writing based on reading a data set which overlaps what the other is writing, you can get a state which could not occur if either had run before the other.

![](https://user-images.githubusercontent.com/17776979/194999902-666afa41-2923-408e-8f69-5291b14e902d.png)

PostgreSQL takes the following strategy for the SSI implementation to prevent write skew:

1. Record all objects (tuples, pages, relations) accessed by transactions as SIREAD locks.
2. Detect rw-conflicts using SIREAD locks whenever any heap or index tuple is written.
3. Abort the transaction if a serialization anomaly is detected by checking detected rw-conflicts.

### Implementing SSI in PostgreSQL

PostgreSQL has implemented many functions and data structures. However, here we uses only two data structures to describe the SSI mechanism

- SIREAD locks
- rw-conflicts,

**Predicate locks (SIREAD lock)**

An SIREAD lock, internally called a predicate lock, is a pair of an object and (virtual) txids that store information about who has accessed which object.

For example, if txid 100 reads Tuple_1 of the given table, an SIREAD lock `{Tuple_1, {100}}` is created. If another transaction, e.g. txid 101, reads Tuple_1, the SIREAD lock is updated to `{Tuple_1, {100,101}}`.

Note that a SIREAD lock is also created when an index page is read.

SIREAD lock has three levels: `tuple`, `page`, and `relation`.

If the SIREAD locks of all tuples within a single page are created, they are aggregated into a single SIREAD lock for that page, and all SIREAD locks of the associated tuples are released (removed), to reduce memory space. The same is true for all pages that are read.

When creating an SIREAD lock for an index, the page level SIREAD lock is created from the beginning.

When using sequential scan, a relation level SIREAD lock is created from the beginning regardless of the presence of indexes and/or WHERE clauses. Note that, in certain situations, this implementation can cause false-positive detections of serialization anomalies.

Predicate locks are not always released immediately on completion of the transaction. It only releases until all concurrent transactions commit

**rw-conflicts**

A rw-conflict is a triplet of an SIREAD lock and two txids that reads and writes the SIREAD lock. The CheckTargetForConflictsIn function is invoked whenever either an INSERT, UPDATE or DELETE command is executed in SERIALIZABLE mode, and it creates rw-conflicts when detecting conflicts by checking SIREAD locks. For example, assume that txid 100 reads Tuple_1 and then txid 101 updates Tuple_1. In this case, the CheckTargetForConflictsIn function, invoked by the UPDATE command in txid 101, detects a rw-conflict with Tuple_1 between txid 100 and 101 then creates a rw-conflict `{r=100, w=101, {Tuple_1}}`

### How SSI Performs

Transactions Tx_A and Tx_B execute the following commands:

![](https://user-images.githubusercontent.com/17776979/195002029-0d15a5f1-0dd6-4db3-894b-5b7f5330e2fe.png)

Assume that all commands use index scan. Therefore, when the commands are executed, they read both heap tuples and index pages.

The image below shows how PostgreSQL detects and resolves the Write-Skew anomaly described in the above scenario.

![](https://user-images.githubusercontent.com/17776979/195002065-3c83e262-fb44-4072-994c-fb24281049b3.png)

### False-Positive Serialization Anomalies

Scenario where false-positive serialization anomaly occurs.

![](https://user-images.githubusercontent.com/17776979/195002465-e52a5c32-4f13-40b3-afb5-f35c7c1ce015.png)

When using sequential scan, PostgreSQL creates a relation level SIREAD lock. In this case, rw-conflicts C1 and C2, which are associated with the tbl's SIREAD lock, are created, and they create a cycle in the precedence graph. Thus, a false-positive Write-Skew anomaly is detected.

![](https://user-images.githubusercontent.com/17776979/195002535-5ba45776-4b19-4e40-9f55-fa975e0b35c4.png)

Even when using index scan, if both transactions Tx_A and Tx_B get the same index SIREAD lock, PostgreSQL detects a false-positive anomaly.

![](https://user-images.githubusercontent.com/17776979/195002557-9f2c58f4-1d8e-4371-900a-90fcb62d44a7.png)
