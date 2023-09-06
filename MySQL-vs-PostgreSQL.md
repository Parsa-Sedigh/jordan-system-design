https://www.youtube.com/watch?v=x6-RN-_i5Xs&t=272s&ab_channel=Jordanhasnolife

SQL DBs are good for relational data. Normalized data where you wanna have one reference to your piece of data and have it stay
consistent throughout all of your data. You don't want to denormalize your data where you'd have to potentially have multiple copies
of a given write and then that would be problematic because you have to write to your data multiple times(if you want to update it, because
it has multiple copies), you might even need distributed transactions if you want to guarantee correctness(consistency) of all of your data.

We want to be using a relational DB when we have relational data that should be normalized. But are there other cases where we should
do it even when we don't care about having relational data? Yes, albeit only sometimes.

Q: Should we ever use relational DBs if we don't need normalized data? Yes, sometimes.

## MySQL vs PostgreSQL

## SQL architecture
### Common features of most SQL DBs
1. B-tree based index -> better for reads?
A B-tree index is shown on the right side of #1 in the image. With B-tree index we have a tree on disk for both reads and writes
that you're directly reading and writing. In contrast to an LSM tree(on the left) and SS table based index where we have memory-based
tree plus a bunch of tables on disk, reads in theory on B-Tree indexes should a little bit faster because in B-tree, we eliminate a whole
branch and we go to read the node we want to, as opposed to an LSM tree based table where we first check the in-memory tree, then check the
on-disk tables(those 3 tables in img).
![](./img/MySQL-vs-PostgreSQL/1.png)

So reads in B-Tree indexes could be faster but writes to B-tree are worse than LSM.

2. single leader replication -> no write conflicts?
Note: You can do any sort of replication with any of these DBs. It just tends to be the case that most people using relational DBs, are using
single leader replication.

When we have multi-leader replication, we can have write conflicts which we can deal with using version vectors or CRDTs but at the end of the
day, these things doesn't necessarily get rid of the conflicts(we would still have conflicts) but they just give us a way of ensuring that
all of our DBs agree on the result. Doesn't necessarily make the result correct whereas with single-leader replication we've got one 
bottleneck, a single DB that all the writes have to go through and thus we can achieve an ordering on our writes.

3. configurable isolation levels -> data correctness? performance cost?
A lot of NoSQL DBs don't necessarily give you the ability to have fully ACID compliant transactions. They're not necessarily completely
isolated and we can't necessarily establish an ordering over all of them as if they're running on one thread.

On the other hand, for relational DBs, you do at least have the option to do that, however it does come at a performance cost obviously,
because it involves either locking or serializable snapshot isolation.

### Isolation differences between mysql and postgresql
mysql: uses two phase locking                           postgresSQL: uses serializable snapshot isolation
- every row has locks                                   - transactions read from data snapshots
- read-only transactions can grab in shared mode        - if transactions reads value which is modified by another transaction before
- to write, must grab the lock in exclusive mode        committing, original needs to be rolled back. 
- lots of deadlocks to detect and undo                   

As we saw, reads in mysql are a little bit more optimized than writes.

Note: So in postgres, there are going to be some transactions that get rolled back, but this is ultimately a form of
OCC(optimistic concurrency control) because we're not locking assuming that everything can go wrong, we're acting as if nothing can go wrong and
then when it does, we roll back

## Conclusions
- use sql DBs both for data that needs to be normalized and for data that needs to be correct. If you need ACID compliant transactions
because otherwise you could have conflicting writes and race conditions, then you need to use SQL DBs. Most of the NoSQL DBs don't support
isolation levels(but they can eventually implement them!)
- in theory SSI > 2 PL, however if there are a lot of conflicting transactions, pessimistic locking(mysql uses it) may be better!
So if there are potentiality a lot of conflicting transactions, it means you're gonna be doing a lot of rollbacks which ultimately might be
more costly than just locking pessimistically.

For example, if everything is like a read-modify-update cycle where those two writes are overlapping with one another, then maybe you'd be better
off with locking than rolling back a lot of your writes. So in this case, the pessimistic locking could be better.

Note: If you care about data correctness you probably want single-leader replication, you probably want ACID transactions and at that point,
a SQL DB mostly makes sense.

Note: The B-tree vs LSM-tree is more of whether you care about prioritizing reads vs writes. 