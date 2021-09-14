# about phantom read

src: 

- https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html 
- https://www.huaweicloud.com/articles/d119b5690eb7daf4b57449a083a76ba1.html
- https://vladmihalcea.com/phantom-read/


Phantom read is defined on MySQL official website as follows:

***The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. For example, if a SELECT is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row***

Phantom read occurs when we combine the operations based on the latest rows (inserting, updating, or deleting) and the normal read backed by MVCC. When inserting, updating or deleting data, the engine must lock the relevant rows to prevent other transactions to interfere the operation. In this case, it always accesses and locks the latest rows/data. 

Differently, when we are simply reading data without locks, MVCC filters out the rows that have version number greater than current transaction, which achieves the isolation between transactions without using locks. But in case when we combine them together, we may see phantom read. 

For example

If we run statement below in transaction **T1**. This statement is executed using MVCC without lock, i.e., we are reading from a snapshot. 

```
1. T1 - START TRANSACTION;
2. T1 - SELECT * FROM user WHERE id < 5;
```

After this, another transaction **T2** inserts a new row with id=3, and if we run the above statement again, we still don't see this newly inserted row, because the REPEATABLE READ isolation level of MVCC prevents this. The newly inserted row has a version number greater than current transaction number, it's not visible by transaction **T1**.

```
3. T2 - START TRANSACTION;
4. T2 - INSERT INTO user (id, status) VALUES (3, 1);
5. T2 - COMMIT; 
```

```
6. T1 - SELECT * FROM user WHERE id < 5;
```

However, when we try to update any rows with id less than 5 in transaction **T1**, we actually updated the newly inserted row (id=3). If we again run the select statement in **T1**, we now see the newly inserted row, or the phantom row, because we now involve updating rows, and our select statement is not backed by MVCC anymore.

```
7. T1 - UPDATE user SET status = 0 WHERE id < 5;
8. T1 - SELECT * FROM user WHERE id < 5;
```

Interestingly, if we now try to insert another row with id less than 5 in transaction **T2**, we will see it being blocked, because in **T1** now holding a gap lock for id < 5. Once we commit **T1**, the insert statement in **T2** will be executed. 

```
9. T2 - INSERT INTO user (id, status) VALUES (4, 1);
```

We can actually prevent this phantom read, if we use **Share Lock** or **Exclusive Lock** in the first select statement in **T1**. In this case, the insert statement in **T2** is blocked by the share lock.

```
1. T1 - START TRANSACTION;
2. T1 - SELECT * FROM user WHERE id < 5 FOR SHARE;


3. T2 - START TRANSACTION;
4. T2 - INSERT INTO user (id, status) VALUES (3, 1); <- blocked 
```

- **Share Lock**

    - A shared lock permits the transaction that holds the lock to read a row, multiple transactions can read the row at the same time, but it prevents write or holding exclusive lock.

- **Exclusive Lock**

    - An exclusive lock permits the transaction that holds the lock to update or delete a row, which is essentially mutex lock.



