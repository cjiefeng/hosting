# MySQL Two-phase commit
Transactions in MySQL using innodb storage engine uses two phase commit (redo log & binlog).

## Binary log
Binary log is a logical log that records any data changes (DML & DDL) in the form of events.

From MySQL reference manual, binary log serves 2 purposes:
> - For replication, the binary log on a replication source server provides a record of the data changes to be sent to replicas. The source sends the information contained in its binary log to its replicas, which reproduce those transactions to make the same data changes that were made on the source.
- Certain data recovery operations require use of the binary log. After a backup has been restored, the events in the binary log that were recorded after the backup was made are re-executed. These events bring databases up to date from the point of the backup.

## Redo log
We write data to disk for persistence. This is a physical log that record modification of physical data pages. Redo logs IO are written sequentially and thus it is faster than writing data which is usually random IO. Redo log is usually represented as `ib_logfile0` & `ib_logfile1` in data directory.

From MySQL reference manual:
> Modifications that did not finish updating the data files before an unexpected shutdown are replayed automatically during initialization, and before connections are accepted.

# Why 2 different logs?
MySQL supports multiple storage engine and redo log is only applicable to innodb. Binary log are usually in statement format so this is independent from storage engine. Theoretically, you can have different storage engine on master & slave and replication will not be affected since replication in MySQL uses binary logs.

# Two-phase commit
1. DML & DDL are first written into redo log in prepare state. (Phase 1)
2. Next is binlog before transactions are committed.
3. Lastly, change status from prepare to commit in redo log. (Phase 2)


Consider situations where we write into one log after another without 2 phase commit:
1. Redo log first, then binlog:  
In the event of MySQL crashing between writing redo log and binary log for a record X. When MySQL recovers, data will be replayed on master for record X but this is not found in binlog. This will result in data inconsistency between master and slave.

2. Binlog first then redo log:  
In the event of MySQL crashing between writing binary log and redo log for a record Y. The opposite will occur where data recovery on master will not have record Y that is available in binary log.

## 2 phase commit to the rescue!
Consider situations where MySQL crashes in between 2 phase commit:
1. Not written into redo log:  
Since changes is not written into redo log, recovery will lose this transaction but redo log & binary log remains consistent.

2. After writing into redo log (prepare state), before writing into binlog:  
because the most recent data is still in prepare state and not found in binlog. MySQL will rollback this transaction to maintain consistency with binary log.

3. After writing into redo log (prepare state), after writing into binlog, before changing into commit state:  
During recovery, when MySQL finds that transaction in redo log is in prepare state and available in binlog and commited, recovery will include this transaction and commit in redo log and recovery completes.  
If transaction is not committed in binlog, transaction will rollback and recovery completes.  

4. After transaction in redo log is in commit state:  
Data is replayed and consistent between redo log and binary log, recovery completes.

As we can see from above, crashing in any moment will result in losing at most 1 transaction. But most importantly data is consistent between redo log and binary log so slaves will remain consistent with master, point in time recovery will also be in a consistent state.

Take note that `sync_binlog` & `innodb_flush_log_at_trx_commit` is set to 1 to maintain this consistency. We will discuss how this 2 will impact data consistency in MySQL in another discussion soon.
