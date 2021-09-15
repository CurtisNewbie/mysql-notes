# InnoDB Storage Engine

src: https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html  

## Basic

To check all available engines:

```
SHOW ENGINES;
```

Or query the `INFORMATION_SCHEMA.ENGINES` table:

```
SELECT * FROM INFORMATION_SCHEMA.ENGINES;
```

# ACID Model

Relevant features for ACID model:

- Atomicity
    - autocommit setting
    - COMMIT & ROLLBACK statements
- Consistency
    - **Doublewrite Buffer**
- Isolation
    - `SET TRANSACTION ISOLATION LEVEL [level]`
        - REPEATABLE READ
        - READ COMMITTED
        - READ UNCOMMITTED
        - SERIALIZABLE
    - Locking 
- Durability
    - **Doublewrite Buffer**
    - **innodb_flush_log_at_trx_commit** config
    - **sync_binlog** config 

# Multi-Versioning

InnoDB add three fields to each row:

- **DB_TRX_ID**
    - 6-bytes field
    - The **transaction identifier** for last transaction that inserted or updated the row, deletion is also treated as an update but the row is marked as deleted
- **DB_ROLL_PTR** 
    - 7-bytes field
    - The **roll pointer** that points to an **undo log record** written to the **rollback segment**. I.e., the pointer points to the undo log records that are able to 'redo' the changes.
- **DB_ROW_ID** 
    - 6-byte field
    - A row ID that increases monotonically as new rows are inserted. It's generated for indexes of rows, however, if primary key is assigned, this value doesn't appear in any index.

**Undo Log** in **rollback segment** are divided into: 

- **Insert undo log**
    - Are only needed for transaction rollback, and thus can be discarded as soon as the transaction commits.
- **Update undo log**
    - Are used for transaction rollback, but are also used for **consistent read** that provides a snapshot of data. 

When rows are deleted, **InnoDB does not remove the row from database physically**, this row is only removed, when the **update undo log** written for this deletion is also discarded, i.e., the consistent read is no longer needed. This deletion operation is also called a **purge**.

# InnoDB Architecture 

- In-Memory Structures
    - Buffer Pool
        - Change Buffer
    - Log Buffer
- On-Disk Structures
    - System Tablespace
        - Change Buffer
    - File-Per-Table Tablespaces
    - Doublewrite Buffer Files
    - General Tablespaces
    - Undo Tablespaces
    - Redo Log 
    - Temporary Tablespaces

[Image of Architecture](https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html)

# In-Memory Structures

## Buffer Pool    

***'The buffer pool is an area in main memory where InnoDB caches table and index data as it is accessed. The buffer pool permits frequently used data to be accessed directly from memory, which speeds up processing'***.

Buffer pool is divided into pages, and it's implemented as a linked list of pages. The pages are expired using LRU (Least Recently Used) Algorithm.

## Buffer Pool LRU Algorithm

The buffer pool is just a linked list of pages, when new page is added to the buffer pool, it's added in the middle of the linked list. The linked list consists of two parts, the new sublist (which contains the young pages), and the old sublist (which contains the old pages). The old sublist is at the tail, so naturally, the old pages are evicted from the tail.

Buffer Pool:

- new sublist (with head)
- old sublist (with tail)

```
head 
 ^
 |   
 | new sublist
 | 
mid
 |   
 | old sublist
 |
 v 
tail
```

When a page is accessed, it then becomes the 'young' page, and it's moved to the head of the new sublist. So, as the database operates, the pages that are accessed less frequently will be slowly moved toward the tail of the list. Eventually, the old page reaches the tail, and gets evicted.

## Change Buffer (In Memory)

Change buffer is a data structure that caches changes to secondary pages resulted from INSERT, UPDATE OR DELETE. Think of it as way to speed up operations to alter data. The changes are merged (to disk) later when the related pages are loaded into buffer pool by other read operations. Recall that change buffer **is part of buffer pool**.

Change buffer and secondary indexes are closely related to each other. The idea is as follows, when we use a normal index that allows duplicates, the changes made can be done in change buffer, because no actual constraint may be violated, the merging can be delayed. 

However, if the index created is UNIQUE index, then we may need to load pages (the UNIQUE index tree) to the memory before we can assure that the change will not violate UNIQUE constraint, then in this case, the change buffer doesn't help much.

I.e., ***Change Buffer doesn't support unique secondary index***. Because we have redo log, in case when the change buffer is not merged or just simply lost, MySQL will recover the changes by looking at the redo log. Notice that, the Change Buffer is also flushed and persisted on disk as well.

Configure **innodb_change_buffering** to change which change operations are buffered:

1. all
    - default value, insert, delete-marking and purge
2. none
    - do not buffer any operations
3. inserts
    - buffer insert operations
4. deletes
    - buffer delete-marking operations
5. changes
    - buffer both insert and delete-marking operations
6. purges
    - buffer the physical deletion operation

**When the change buffer merging occur**:

- When a page is read into buffer pool
- Periodically by a background task 
- During crash recovery
- During system restart

## Log Buffer

***The log buffer is the memory area that holds data to be written to the log files on disk.*** The size of log buffer can be changed by **innodb_log_buffer_size**, the default is 16MB.

The contents of log buffer are periodically flushed to disk. If the log buffer is large enough, it allows recording large transactions without writing redo log to disk before the transactions commit.

Configure how log buffer are written and flushed to disk using **innodb_flush_log_at_trx_commit**:

- 0 
    - logs are written and flushed once per second
- 1
    - default, logs are written and flushed at each transaction commit
- 2
    - logs are written and flushed once per second and at each transaction commit

Configure writing and flushing logs every N seconds using **innodb_flush_log_at_timeout**:

- default value is 1, the permitted range is from 1 to 2700

# On-Disk Structures

## Physical Structure of an InnoDB Index

Index records are stored in leaf pages of B-trees (on the bottom), the **default page size** is 16KB, and it's determined by **innodb_page_size**. When new records are inserted, InnoDB tries to leave **1/16 of the page free** for further insertion or updates. 

If the index records are inserted **sequentially**, the pages are about **15/16 full** leaving the 1/16 free. However, if the records are inserted in **random order**, the pages are **from 1/2 to 15/16 full**.

## System Tablespaces

***"The system tablespace is the storage area for the change buffer. It may also contain table and index data if tables are created in the system tablespace rather than file-per-table or general tablespaces."*** Without configuration, the tables are always created in **File-Per-Table tablespaces**.

- Change Buffer
- table and index data
    - if it's configured to store table files in System Tablespaces, by default, it uses File-Per-Table Tablespaces

## File-Per-Table Tablespaces (Default)

***"A file-per-table tablespace contains data and indexes for a single InnoDB table, and is stored on the file system in a single data file."***

Advantages:

- Disk space returned to OS after truncating or dropping table in file-per-table tablespace, but a shared tablespace data file doesn't not shrink after a table is truncated of dropped.
- Better **TRUNCATE TABLE** performance.
- Data files may be stored in external storage.
- Data files are easier to backup and have higher change of a successful recovery when data corruption occurs.
- Concurrent write to different tables/data files.
- Avoid size limit for a single shared data file.
- ...

Disadvantages:

- **fsync** operation per file/table, leading to higher total number of **fsync** operations.
- More file descriptors
- More open file handles
- ...

## General Tablespaces

***"A general tablespace is a shared InnoDB tablespace that is created using CREATE TABLESPACE syntax."***

Capabilities:
- Capable of storing data for multiple tables
- May consume less memory for tablespace metadata compared to File-Per-Table Tablespaces
- Data files may be stored in external storage.
- ...

Limitations:

- Not support temporary tables
- Space freed by truncating and dropping tables is not released back to OS
- ... 

## Undo Tablespaces

***"Undo tablespaces contain undo logs, which are collections of records containing information about how to undo the latest change by a transaction to a clustered index record."***

Two default undo tablespaces are created when MySQL instance is initialized. They are to provide a space to store the **rollback segments**. 

Recall that undo logs are also used to provide consistent view. So a undo tablespace is not dropped until it's marked as inactive, i.e., any transactions started before those committed transactions are completed, the consistent view / snapshot for these rows are not needed anymore. Then, these **rollback segments** are purged, and the undo tablespace is truncated to its initial size.

## Temporary Tablespaces

- Session Temporary Tablespaces
    - ***"Session temporary tablespaces store user-created temporary tables and internal temporary tables created by the optimizer when InnoDB is configured as the storage engine for on-disk internal temporary tables."***
- Global Temporary Tablespaces
    - ***"The global temporary tablespace stores rollback segments for changes made to user-created temporary tables."*** 

## Doublewrite Buffer

***"The doublewrite buffer is a storage area where InnoDB writes pages flushed from the buffer pool before writing the pages to their proper positions in the InnoDB data files. If there is an operating system, storage subsystem, or unexpected mysqld process exit in the middle of a page write, InnoDB can find a good copy of the page from the doublewrite buffer during crash recovery."*** By default, there are two doublewrite buffer files per buffer pool.

## Redo Log

***"The redo log is a disk-based data structure used during crash recovery to correct data written by incomplete transactions. During normal operations, the redo log encodes requests to change table data that result from SQL statements or low-level API calls. Modifications that did not finish updating the data files before an unexpected shutdown are replayed automatically during initialization, and before connections are accepted."***

Redo log is written in a **circular fashion**, the logs are identified by an ever-increasing **LSN Log Sequence Number** value, which represents a point in time. The **LSN** is mainly used for crash recovery and management of buffer pool.

**Group Commit for Redo Log Flushing**

- By default, InnoDB **flushes redo log of a transaction before it's committed**. However, the **commits are grouped and flushed together** to avoid one flush for each commit.

## Undo Logs

***"An undo log is a collection of undo log records associated with a single read-write transaction. An undo log record contains information about how to undo the latest change by a transaction to a clustered index record. If another transaction needs to see the original data as part of a consistent read operation, the unmodified data is retrieved from undo log records."*** I.e., the undo logs are also used to provide a consistent view or snapshot view.

**Structure and hierarchy**:

- Undo tablespaces and Global temporary tablespaces
    - stores Rollback segments
        - contains Undo log segments
            - contains Undo log 


#  Locking and Transaction Model

## InnoDB Locking

### Shared and Exclusive Locks

- A **shared (S) lock** permits the transaction that holds the lock to read a row, there can be multiple holders of the lock, and the exclusive lock is granted when a shared lock is hold.
- An **exclusive (X) lock** permits the transaction that holds the lock to update or delete a row, it's mutex lock, thus when the exclusive lock is hold, both shared lock and exclusive lock are not granted until the lock is released.

### Intention Locks

***"Intention locks are table-level locks that indicate which type of lock (shared or exclusive) a transaction requires later for a row in a table."*** E.g., locking specific rows for later update, which then prevents other transactions to update the lock rows, even though current transaction doesn't actually update these rows. It describes the 'intention'.

- **Intention Shared Lock**
    - The shared lock on rows in a table
    - e.g, SELECT ... FOR SHARE
- **Intension Exclusive Lock**
    - The exclusive lock on rows in a table
    - e.g, SELECT ... FOR UPDATE

***"The main purpose of intention locks is to show that someone is locking a row, or going to lock a row in the table."***

### Record Locks

Record lock is simply a lock for an index record. In case when the table doesn't explicitly define a primary key, a hidden index is generated that will still be locked.

e.g., the statement below locks the record with the primary key 'id' that equals 1.

```
SELECT * FROM t WHERE id = 1 FOR UPDATE;
```

### Gap Locks

***"A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record"*** If a range is specified, all values within the range is locked by the gap lock.

e.g., the statements below lock rows with c1 between 10 and 20.

```
SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;
```

### Next-Key Locks

***"A next-key lock is an index-record lock plus a gap lock on the gap preceding the index record"***

### Insert Intention Locks

An Insert Intention Lock is set prior to row insertion. This works essentially the same as the intention locks mentioned above. If another transaction T2 holds a shared gap lock (e.g., id > 5) that includes the index value that the current transaction is trying to insert (e.g., id = 6). The current transaction T1 will block when it tries to obtain the insert intention lock, it has to wait until the shared lock being release by the other transaction T2.

### AUTO-INC Locks

This is a table-level lock applied to AUTO_INCREMENT for primary key.

## Transaction Model

- REPEATABLE READ
    - Consistent read / snapshot backed by MVCC
    - Lock
        - Locking reads
            - SELECT ... FOR UPDATE
            - SELECT ... FOR SHARE
        - UPDATE 
        - DELETE
- READ COMMITTED
    - Consistent read / snapshot backed by MVCC
    - Lock
        - Locking reads
            - SELECT ... FOR UPDATE
            - SELECT ... FOR SHARE
        - UPDATE
        - DELETE
    - Gap Lock is disabled only records are locked, thus Phantom Read may occur 
- READ UNCOMMITTED
    - No consistent read
    - Non-locking read, thus may occur dirty read
- SERIALIZABLE
    -  REPEATABLE READ, but with all plain SELECT statements being converted to 'SELECT ... FOR SHARE' 

## Consistent Non-locking Reads

***"A consistent read means that InnoDB uses multi-versioning to present to a query a snapshot of the database at a point in time. The query sees the changes made by transactions that committed before that point in time, and no changes made by later or uncommitted transactions. The exception to this rule is that the query sees the changes made by earlier statements within the same transaction."***

Notice that the current transaction will see changes made by statements within the same transaction, this is what may cause the **Phantom Read**. Thanks to MVCC, we don't see changes 'after' current transactions, but if we somehow changed these rows, we are now be able to see them.
