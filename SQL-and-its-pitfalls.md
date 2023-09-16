https://www.youtube.com/watch?v=coJNGQFJ5kM&ab_channel=Jordanhasnolife

Not all relational DBs use SQL.

## Relational databases background
- tables holding many rows of structured data(pre-defined schema)
- rows of one table can have a relation to rows of another if they share a common key(foreign key)
- built-in query optimizer that returns results using the declarative SQL language

Popular RDBMS: MySQL, PostgreSQL

### Other implementation details
- generally, they use B-Trees
- support transactions with 2 phase locking for isolation
- all reads and writes go to disk

## scaling a relational database
- vertical scaling
    - increasing the power of the hardware running the database(traditional approach)
- horizontal scaling
    - adding more computers/nodes of similar power, distributing workload
    - means that we have to shard/partition our dataset, this is where things get hard

## the problems with sharding
Imagine that we have a single-leader replication setup for each shard. We want to do a transaction between different partitions.
Note that we're no longer just using a single computer level transaction, now we're doing a distributed transaction.
![](./img/SQL-and-its-pitfalls/1.png)

Since distributed transactions have to get both(or more) of the nodes to agree on doing sth, is going to use a lot of network resources
and it's gonna be much slower than single computer transaction. Therefore, writing to multiple partitions at once in a transaction way,
can be very slow.

## the problems with sharding continued
![](./img/SQL-and-its-pitfalls/2.png)

We have to query both partitions to get the result we want.

The fact that we have to make multiple network calls is problematic. Why? Because one of the network calls could fail and then we would have
an incomplete query. Or the fact that we have to do multiple of this query means things are gonna take longer.

Network reqs are always bad and you wanna minimize them.

## the relational philosophy
This philosophy in many ways can scale poorly.

- one copy of every piece of data(reduce duplication). We want to normalize data not de-normalize it, that way if we make a change to one
piece of data which a bunch of other pieces of data reference, that change is now going to be propagated. This is the idea of reducing
duplication.
- each table has one preset schema: This allows you to kinda encode data better
- fetch related data via joins
- hide concurrency bugs and partial failures via transactions(a form of abstraction)

### Issues with relational databases at scale
- splitting related data up over partitions/different tables, becomes very problematic once network delay becomes involved
  - the need for checking each partition or using distributed transactions greatly slows things down. The second the sharding gets
  involved, all of this data splitting becomes problematic, we have to make a bunch of network calls both on reads and writes. On reads,
  on reads for aggregating results from partitions and on writes for distributed transactions
- to have the transactions even on a single node, the locking needed by transactions in order to enforce isolation is too slow.
  This requires two-phase locking and it can be slow because it means reads and writes can block one another
- B-Trees are very slow for writing compared to some in-memory buffer like an LSM tree

## moving away from relational databases
The term NoSQL has arisen as a general term for any DB that is not both relational and using the SQL query language.

In reality, NoSQL DBs are more stripped down than relational DBs and give the developer more opportunists to choose one that
fits the needs for their application, because sometimes it's better at huge scale to abandon some of the features of relational DBs
in exchange for greater performance.

So NoSQL DBs aren't the opposite of SQL or anything like that, it's just taking a SQL DB and stripping it down some of it's features
and abstractions which are a black box.

## NoSQL design patterns
Most NoSQL DBs are self-contained documents, there are also NoSQL DBs that are graph DBs nad graph DBs are good for many to many
relationships which those document DBs are not sufficient for.

Document DBs are generally good for when data is in self-contained documents and there's not too many relations between the data.

Graph DBs represent things like social networks really well, because they allow you to put metadata with each node and link between the nodes.

- most importantly, objects are generally self-contained documents.
  - more locality on disk for both reads and writes(good for when accessing whole document). Everything that is related in data,
  is gonna be stored next to one another. For example in linked profile, we have some job experiences for a user. Instead of storing
  the job experiences in their own table(collection or ...), we would store those experiences in a JSON document with the user profile.
  So generally, everything is gonna be stored together. This is good because sequential reads on disk are more efficient and as a result,
  everything is going to be sequential and easier to read from and write to. This also makes things easier to shard
  - easier to shard. We can literally just split up the documents, it doesn't matter which shard they're on, because at the end of the
  day, everything that that document is going to need(in terms of aggregating data) is **already** with it(it's self-contained)
  - schemaless
    - This is really good for applications like ML. Anytime you just wanna dump a ton of data into a DB without worrying about
      formatting it or putting it in a way that it needs to fit into a given table.
    - It's also good for maintainability. Because that means that your data can adapt over time in the event that you want to make
    changes to your data without having to add things like optional fields.
  - the main **pitfall** of all of this is: data duplication(needs to be handled in application code). Example: Let's say we're using
  a NoSQL DB and we have documents for the books in the library. In an SQL(relational) DB, each author would have a row in the table and if
  an author writes multiple books, each book can reference the same author via an author_id. In a NoSQL DB, we might have to
  repeat that author information multiple times. Now let's say this author is still alive and decides to change their name or ... .
  Now we have to go to every single book written by that author and change their name. So this could potentially take a lot of
  application side code which is not good
- graph databases
  - good for many to many relationships, everything can be related.
  - schemaless

## relational databases conclusion
- very intuitive data model that is easy to understand
- however, tend to scale poorly when sharded(or when you wanna do some sort of horizontal scaling)
    - on writes to many shards may need distributed transactions
    - on reads to many shards involves many network calls(because we have to aggregate data across multiple partitions)
- transaction abstraction and locking is slow(two phase locking can be slow)
- B-Trees are slow for writes(since they go to disk). Compared to things like LSM trees which go to memory
- set schema. The set schema makes the DB hard to maintain in the long run in the event that your data is evolving
- generally use single leader replication, because it's simpler. As a result of that, it limits your write throughput. So for
write throughput increases, it tends to be the case that you wanna use a NoSQL DB

As a result, many developers have chosen to use "NoSQL" DBs, which relax some of these requirements and diverge from the data patterns of SQL,
in the hopes of improving performance for their application. We will examine some of the said DBs in subsequent videos. But this will
add some complexity to your code.

For example if you're a bank and you wanna make sure account balances are consistent and synchronized, you do need transactions and relational DBs.
But in for example messanger services and social media apps, the increased write throughput and lack of need for complete consistency by NoSQL,
is a good thing.

With all this said, this doesn't mean that SQL is infeasible. For many read-heavy applications with highly related data, relational DBs are a good
solution. Many companies(see VoltDB, google spanner) have even tried to improve the scalability of the relational model, which some have
dubbed NewSQL. In other words, many companies like the SQL interface but wanna improve the underlying technology below it.

There is no one size fits all database, and it is only by knowing how each one of these popular DBs work that we can determine which to use for
a given application.

---

**Note:** Small correction on "B trees are slower for writes compared to LSM trees which go to memory". Even in LSM trees, writes are usually 
written to disk in a write ahead log for durability. But the fact that it is a simple **append** operation on a file is what makes it fast compared to
B trees where you have to do a log(N) operation and change the exact block where it's contained.