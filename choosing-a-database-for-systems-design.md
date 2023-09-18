https://www.youtube.com/watch?v=6GebEqt6Ynk&ab_channel=Jordanhasnolife

## Choosing a database
Often times it's better to use more than one type of DB.

## review
### Background
There are a lot of different options for DBs and each of them has a unique implementation! How can we decide which to choose for our
system design interview?

### Review: Indexes
A database index is used for the purpose of speeding up reads conditioned on the value of a specific key! Be careful not to overuse indexes, as
they slow down DB writes.

In today's DBs that are on disk, there are two main types of DB indexes:
- LSM trees + SSTables
- B-Trees

### Review: LSM trees and SSTables
We have our LSM tree which is a balanced binary search tree in memory(but still there's a write-ahead log on disk for durability). You can
have a LSM tree with sth like a red black tree. This tree has keys and their corresponding values. When that tree gets too big, it gets
flushed to a SSTable on disk.

This LSM tree is also called memtable.

A SSTable is effectively just a sorted list of keys and their values on disk.

When we wanna read a key, we first check for it in the memtable(that LSM tree) and then if it's not in there, we go through 
the SSTables in order from the most recent one to the least recent one. So we're never overwriting any keys, we just write new values for
those keys and we keep creating new SSTables.

If we're running out of space, we can compact our SSTables by merging them together, it's kinda like the merge sort algo where you're merging
two sorted lists and then that way we can free up some space again.

Note: Because our SSTables are **sorted**, we can kind of keep a sparse index of the location of some keys in the SSTable and that allows us to do
searching really fast in O(log n) via a binary search.

Pro: we have fast writes because those writes are going to an in-memory buffer in O(log n) because of being a balanced BST - they are also
written to a write-ahead log on disk which would be O(1)

Cons: For reads we potentially have to go through many SSTables to find the value of a key. We can use things like bloom filters to
speed this up, but ultimately it's just not as good as B-Trees for reading.

![](./img/choosing-a-database-for-systems-design/1.png)

### Review: B-Trees
B-Tree is also a binary search trees and it's on disk. Basically we have pages on disk that have a range of keys and a bunch of pointers.
So that you can figure out for a specific value of a key, you iterate through a few of pointers and then go to the correct place on disk.

Pros: Better reads because we don't have to iterate through a bunch of SSTable files(even with bloom filters, they're still slower in reads), we can
go directly to the location of the key by following the B-Tree on disk

Cons: Writes in an LSM tree are going to memory, LSM trees are better for writes.

B-Trees are worse for writes but faster for reads

![](./img/choosing-a-database-for-systems-design/2.png)

### Review: replication
Replication is duplicating the data you have on one DB node to another DB node, so that in the event of some hardware or software failure,
you can make sure that data is retained.

Replication is the process of having multiple copies of data in order to make sure that if a DB goes down, the data isn't lost!

Types:
- single leader replication: You have one master node that writes to the others.
    - all writes go to one DB, reads come from any DB
- multi leader replication
    - writes can go to a small subset of leader DBs, reads can come from any DB
- leader less replication: You may use some sort of majority voting process to decide what the accurate read is?
    - writes go to all DBs, reads come from all DBs

### Review: Replication continued
We don't have write conflicts in single-leader replication

Single leader:

Cons: has lower write throughput because all of the writes have to go through that one master node. You can try mitigate this with partitioning
or sharding but at the end of the day, if all the writes have to go through one partition(?), thr throughput is gonna still be limited.

Leader less and multi leader:

Technically, we have unlimited write throughput in terms of we could be writing to different replicas every single time. However,
at the same time, the more replicas that we write to, the more likely there is to be a write conflict. There are smart ways for avoiding this,
like certain writes going to the same replicas, but at the same time, it's good to be able to know that your data is correct and you can
avoid conflicts at all times.

![](./img/choosing-a-database-for-systems-design/3.png)

## SQL databases
The feature set is generally similar between SQL DBs. So breaking these even more is not important. Because they're all based on the same
technologies, but they may have subtle differences in terms of feature sets.

Key features:
- relational/normalized data(data model) - changes to one table may require changes to others. We don't duplicate data in multiple places,
but rather you'll use joins to reference pieces of data with one another throughout tables. What this means is that a lot of times when you're
doing writes, you may have to write to multiple different tables and if those tables are in multiple different nodes, that can be a problem
at scale. The reason is if you wanna write to two different tables and they're on two different computers, for those to either both succeed
or fail, we would need sth like a two-phase commit.
  - E.g. adding an author and their books to different tables on different nodes
  - may require two phase commit(Expensive)
- have transactional(ACID guarantees)
  - excessively slow if you don't need them(due to two phase locking)
- typically use B-Trees
  - better for reads than writes in theory(compared to LSM trees + SSTables)

**Note:** Two phase commit means a transaction over multiple computers(nodes). Distributed transactions are unreasonable in practice and as a result,
it's hard to implement.

**Note:** The issue with transactions is even though it's good for correctness of the data, it also potentially slows down the entire DB due to
most of the SQL DBs using an expensive two phase locking scheme in order to ensure data correctness.

**Conclusion:** Use SQL when correctness is more of importance than speed
- see banking applications, job scheduling.

In job scheduling, we could have a status table and we're editing multiple rows of that table at the same time to do updates, we wanna make sure
that those are relatively in sync and all of those rows are either being changed or not.

## mongoDB
Key features:
- document data model(NoSQL)
  - data is written in large nested documents, better data locality(if you choose to organize your data in a way that takes 
  advantage of this) - but denormalized. Denormalized means in this case means you have multiple docs that rely on same piece of info,
  if one piece of info gets updated but the other doesn't, then those docs are out of sync and that's the issue with denormalized data.
- B-Trees and transactions supported

**Conclusion:** Rarely makes sense to use in a system design interview since nothing is "special" about it, but good if you want SQL
like guarantees on data with more flexibility via the document model

## Apache Cassandra
You should probably be listing this in most of your system design interviews.

key features:
- wide column data store(NoSQL), has a shard key and a sort key
  - allows for flexible schemas, ease of partitioning
- multileader/leaderless replication(configurable): So you can choose sth like quorum writes, but at the same time you can also just
choose to have one node be the leader and propagate that out elsewhere or have a couple of nodes be the leaders but it's not necessarily
going to be a quorum read or write, so you don't always need the majority
  - super fast writes, albeit uses last write wins for conflict resolution
  - may clobber existing writes if they were not the winner of LWW
- index based off of LSM and SSTables
  - fast writes, slower reads(compared to B-trees)

Note: wide column data store is essentially an excel spreadsheet. Other than the required sharding and sorting columns, you can
put any other columns in there and that can change from row to row, which is nice in terms of having data flexibility.

Note: LWW for resolving the write conflicts isn't always ideal, because it means certain writes can be clobbered if they had the lower timestamp.
Keep in mind that timestamps in distributed systems are not reliable unless you're using sth like a GPS clock.

Conclusion: Great for applications with high write volume, consistency is not as important, all writes and reads go to the 
same shard(no transactions).
So if it's ok to occasional piece of data is overwritten or lost, cassandra is great.

- see chat application for a good example of when to use. Like facebook messanger chat where your sharding key would be the chat id and then
from there, you would have all of your messages ordered using timestamp as the sort key

## Riak
key features:
- Key-value store(NoSQL), has a shard key and a sort key
  - allows for flexible schemas, ease of partitioning
- multileader/ leaderless replication(configurable)
  - super fast writes, supports CRDTs(conflict free replicated data types)
  - allows for implementing things like counters and sets in a conflict free way, custom code to handle conflicts
- index based off of LSM and SSTables
  - fast writes, slower reads(compared to B-trees)

Same use cases as cassandra.

Note: CRDTs act as ways of aggregating certain conflicting writes and then allowing you to use more complex logic in order to resolve the conflicts.
This is useful if you wanna make sure that you're not having writes clobbered and it can also allow you to create things like sets and counters
in a multileader or leaderless replication setup that are eventually consistent.

**Conclusion:** Great for applications with high write volume, consistency is not as important, all writes and reads go to the same
shard(no transaction)

- see chat application for a good example of when to use

## Apache HBase
It's a wide column store, kinda outer interface as cassandra. However there are some subtle differences that do make a difference in performance.

The interesting thing about HBase is that the actual storage itself instead of using a row-wide storage model, uses column-wide storage which means
instead of storing data within the same row together, you store data within the same column of the table together and as a result,
you can achieve much better data locality when you're trying to read the entirety of just one column.

key features:
- Wide column data store(NoSQL), has a shard key and a sort key
  - allows for flexible schemas, ease of partitioning
- single leader replication
  - built on top of HDFS or hadoop distributed file system, ensures data consistency and durability
  - slower than leaderless replication
- index based off of LSM and SSTables
  - fast writes, slower reads(compared to B-trees)
- column oriented storage
  - column compression and increased data locality within columns of data

HDFS means you're using single leader replication. You write to a single node(leader) and then from there that data gets replicated over other nodes.
So this is gonna be slower than leader less replication but at the same time it means we don't have to worry about write conflicts.

**Conclusion:** Great for applications that need fast column reads. For example in tiktok app, when you click on a video, it shows you a few images
of that video. That would be a great place to use HBase, because you can store that raw image data in the table itself and use column compression
to quickly pull all of those images from that column. Whereas if it was row storage, we would have to read multiple rows, you would have less data
locality and the read would be slower.

- multiple thumbnails of a youtube video, sensor readings

## Memcached and redis
Instead of using anything like an SSTable + LSM tree combo or B-Tree, you simply use a hashmap to store your data, so you don't need an index.

Note: Hashmaps are worse for range queries.

The point about memcached and redis is that they're useful for data where it's gonna be written and read very frequently.

key features:
- key-value stores implemented in memory(redis is a bit more feature rich)
  - uses a hashmap under the hood

Conclusion: Useful for data that needs to be written and retrieved extremely quickly, memory is expensive so good for small datasets

- good for caches, certain essential app features(see geo spatial index for Lyft). So one example is the geospatial index in an app like
uber or lyft where it's constantly being updated and read from with all the updated positions of drivers and riders.

## Neo4j
The point of graph DB: If we wanted to implement a graph with a SQL DB, we could do it but we would have to be using a many to many relationship and
the issue with many to many relationship is we would have a table with ids of the nodes that have relationship. So the fact that two nodes were in that
many to many table, would indicate that there's an edge between them. So we could do that, however that's gonna be very slow. Because as that table
gets bigger, we know our index for that table is gonna get bigger and note that the time complxity of reading from an index is proportional to O(log n).
n is the number of elements in the index. So as the DB table grows bigger, our graph DB would get slower if we implemented using a SQL table.
Instead, neo4j is a native graph DB which means you have pointers to the actual location of the node corresponding to the other end of the edge on disk.
So you can more quickly traverse them(in constant time(O(1))) as opposed to having to do O(log n) every single time you want to traverse a node.

key features:
- graph database!
  - as opposed to just using a SQL database under the hood with relations to represent nodes and edges, actually has pointers from one address
  on disk to another for quicker lookups
  - the former is bad because reads become slower proportional to the size of the index(O(log n) to binary search), but using direct pointers is O(1) 

Conclusion: Only useful for data naturally represented in graph formats. Niche.

- Map data, modeling our friends on social media

## Time series
TimeScaleDB/ Apache Druid

Timeseries DBs are useful for when you're modeling a bunch of ingestion for many different sources of data, but also wanna keep them in order relative
to their timestamp. They use LSM Trees + SSTables so you can have relatively fast ingestion because things go to that in-memory buffer first, however
they also take a little bit of a turn on them, to make them more efficient. For starters, as opposed to having one huge LSM index for the entire table,
what they do instead is split the table into many small indexes. The reason is you're likely only going to need to be able to access
a small chunk of data at a time in a time series DB and being able to make these indexes into a small chunk indexes, is going to allow you
to put that entire index in memory(the SSTable) and as a result, quickly read from it and so you can use the CPU cache efficiently by having
all of these smaller indexes.

Additionally this is efficient for deleting those indexes. Because a lot of times with timeseries DBs as the data gets too old, you wanna throw it out.
But this is inefficient with a typical LSM index where you wouldn't actually delete the data directly, but you would put in a tombstone into the
LSM tree where you're saying that key is going to be deleted and then eventually when those SSTable files get merged together, that key is going to be
thrown out. However, by actually deleting the index itself, this is a much faster process than having to wait for the table compaction to occur.

key features:
- time series database!
  - use LSM trees(and SSTables) for fast ingestion, but break table into many small indexes by both ingestion source and timestamp
  - allows for placing the whole index in CPU cache for better performance, quick deletes of whole index when no longer relevant(as opposed to
  typical tombstone method)

Conclusion: Very niche but serve their purpose very well

- Great for ingesting tons of sensor data, metrics, logs, where you want to read by the ingestor and range of timestamp

## Honorable mentions
### New SQL
#### VoltDB
Means they're SQL DBs, however they're implemented in interesting ways to get better performance than traditional SQL DBs.

Since in voltDB there's only a single thread, there can't be any race conditions. By running it all in memory, we can make these operations happen so fast that
using single threaded execution is actually feasible for performance.

#### Spanner
The premise of spanner is using SQL and in order to avoid having to do a ton of locking to figure out what write came after what write,
what does it instead, is it puts GPS clocks in the actual datacenter itself in order to assign each write a relatively accurate timestamp and then
use those timestamps to figure out what write came after what write and once you can order all of these writes, it means that all of the DBs are able to
go in a consistent state and we can achieve strong consistency by virtue of linearzability.

- voltDB
  - SQL but completely in memory, single threaded(of CPU OFC!) execution for no locking
  - expensive(because running in memory) and only allows for small datasets(because running in memory)
- spanner
  - SQL, uses GPS clocks in data center to avoid locking by using timestamps to determine order of writes
  - very expensive - because we're no longer using commodity hardware, we're actually putting clocks in the datacenter which is not an easy thing to do

Cool discussions to have in an interview, but probably pretty impractical to ever actually suggest!

Data warehouses: These are generally for SQL formats and probably after you would do some sort of batch work to properly format the data.
They are for analytics.