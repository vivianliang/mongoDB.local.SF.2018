# How and when to use multi-document transactions
**speaker: alyson cabral - product manager, distributed systems.** (replication, sharding, etc.)
@aly_cabral

this talk was very similar to this blog post: https://www.mongodb.com/blog/post/mongodb-multi-document-acid-transactions-general-availability

multi-document ACID transactions introduced in MongoDB 4.0

## why do we need multi-document transactions?

in a relational database, transactions must span across multiple tables.

with mongo, we should be modeling our data in a fundamentally different way (related data is stored together in a single document). if we're following best schema design principles, most~ of the time, we should not need multi-document transactions.

however, sometimes we do!

example multi-document transaction use cases:
- positions and trades
    - infinite number of trades and have to update position every time
- account creation
- application event logging: auditing
    - ex: update ownership of google doc while updating log event of ownership change

## transactions 

Example of transactions API:
```python
    with client.start_session() as s:
        s.start_transaction():
        try:
            owners.insert_one(doc1, session=s)
            owners.insert_one(doc2, session=s)
        except:
            ... 
        s.commit_transaction()
```

snapshot isolation: transactions have consistent view of data, all or nothing execution.

read your own writes: 
- transaction can read its own uncommitted writes. other operations outside of the transaction see the old/uncommitted value.

if a transaction occurs while another transaction has write lock for document, second transaction is aborted and rolledback entirely.

## write conflicts

- typical writes, before mongo 4.0:
    - not wrapped in multi-document transaction
    - operation blocks. infinitely retries with backoff logic until MaxTimeMS

- when a transaction encounters a write conflict, it will abort and rollback
- if a non-transactional write encouters a lock being held by a transaction, the operation  will block and retry until MaxTimeMS

- reads don't take a lock. if the document being read has uncommitted writes, we'll just be reading the committed values.

- only writes trigger write conflicts. noop writes (setting a field to the same value it had) are optimized away, so a write isn't actually performed and won't trigger a write conflict. to guarantee we get a write conflict, change something, i.e. increment a counter.

## replicas
when "commit" occurs, primary writes are written atomically to secondaries.

write concern: determines when server sends the success message--how many nodes does it need to commit to?
    - the default is w=1--once the primary has committed.
    - alternatively w=majority--once the majority of nodes have committed. <-- guarantees we will never lose the write.

multi-document transaction with w=majority is faster than 100 writes each with w=majority. pays the latency cost only once.

## WiredTiger Cache
- cache pressure builds up by write volume after a snapshot
- transactions use the same snapshot throughout it's duration
- writes can't be flushed until transactions running on old snapshots commit or abort

how to avoid WireTiger cache pressure:
- transactionLifetimeLimitSeconds = default to 60s
- commit read-only transactions
- abort abandoned transactions (transations are not free. have to maintain snapshots)
- <1000 documents modified within a transaction
- transaction is represented in single oplog entry, so should be <16mb document size limit

DDL Operations, i.e. createIndex, createCollection, dropDatabase
- DDL operatiosn block behind transactions
- multi-document transactions take intentlocks for longer
- if there is a pending DDL operation, transactions are prevented from obtaining new intent locks and abort.

## Key Points

1. all data modeling rules still apply
    - don't use it like a relational database!
    - store related data togther, as direct business entities
2. transactions shouldn't be most common operation of your application (sanity check for rule 1)
3. pass in the session to all statements
4. implement retry logic, transactions can always abort
    - be able to seamlessly recover
5. don't unnecessarily leave snapshots open. they are not free. maintains history in cache
6. to trigger write conflicts, make sure you're doing writes
7. plan for DDL Operations
