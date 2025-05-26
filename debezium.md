- impls change data capture
- supports many different source & sink dbs

### What is CDC?
If I'm a client and I'm writing to some sort of source DB, CDC allows us to stream those writes(changes) to a bunch of sink technologies.
Sink techs could be elasticsearch, some sort of data warehouse, flink ... . So writes to a source DB, are ingested to downstream sink systems.
![](img/debezium/1.png)

### Naive(bad) CDC implementations
1. pull based: pull the DB on an interval
    - each sink polls the DB on an interval to detect new changes
    - extra work for DB(have to handle a lot of read queries, maybe there aren't new changes so the poll was wasted and ...)
2. DB trigger based
    - DB triggers run after each operation to push it somewhere
    - extra work for DB
    - potential consistency issues depending on how you use triggers

How does debezium impl CDC?

Idea: Log based CDC

We know every DB for fault tolerance or for replication, has some sort of log of writes that it maintains.

Depending on the log being for write-ahead or for replication, there has to be different impls of debezium for every single
source DB.

- hook into existing write-ahead/replication log and stream the changes to sink DBs
- takes the changes and converts them into DB agnostic format and push them to sink DBs

What does DB agnostic format looks like?

Some json like this:
```json
{
   "type": "create/update ...",
   "fields": ["(name, string, jordan)", "(..., ..., ...)"]
}
```

For updates, instead of having before & after vals, just include the diff to make the msg smaller.

### Debezium message delivery
It operates in 2 places:
- as source connector
- as sink connector

Source connector continuously reads the write-ahead/replication log of the source DB and then publishes them to kafka.
Then sink connector reads the data out of kafka and sends it to sink technologies.

Both connectors are using kafka connect.

Q: Why we use kafka here as opposed to not having anything in the middle at all(for storing the db agnostic ops) and just **directly** publishing
from source connector to sink techs?

A: For 2 reasons:
1. Kafka makes this entire process asynchronous which is good because the write ops could be expensive and hard-to-process.
2. we get `at-least-once` msg delivery. Since kafka is a log-based msg broker, all of those write ops are gonna be persisted at least for
some amount of time and if anything fails at the sink connector, we can re-read the event from kafka and retry it. Note that all these
events(write ops) should be handled idempotently, because we're dealing with **DB** writes. So by having kafka, the producers and consumers
are decoupled. The producers(source connectors) are responsible for transforming the writes into db agnostic format and sink connectors are responsible for
reading the agnostic writes and sending them to sink techs.

Q: What are some sources and sinks that debezium actually supports?

Sources:             
- mysql
- mariadb
- mongodb
- postgresql
- oracle
- sql server
- db2
- cassandra
- vitess
- spanner
- informix

sinks:
- any db that works with jdbc

### Data snapshotting
Since debezium is log-based, when it's working with a write-ahead log, we know that eventually write-ahead log is gonna get compacted.
Because the WAL can't grow infinitely in size otherwise your db will run out of storage. So eventually the WAL 
is gonna get compacted by squeezing writes to the same key and just keep the most recent ones.

So now how does debezium can find the data it needs from compact log? Because let's say we have a db that already has 1 million writes
and therefore the writes are already been compacted, now debezium doesn't have access to old writes. It just has the last result of
writes, not previous changes(writes).

What debezium will do is send a full table snapshot or partial snapshot of table to kafka.

### CDC use cases