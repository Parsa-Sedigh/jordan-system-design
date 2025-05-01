### At least once / at most once delivery
For both producers & consumers, you can configure them as either at least once or at most once in terms of how they process msgs.

- at least once: 
  - producer: retry producing msg when acknowledgment not received from broker
  - consumer: do not commit msg offsets until msg is fully processed
- at most once:
  - producer: do not retry producing msg when acknowledgement not received from broker
  - consumer: commit msg offsets **before** msg is fully processed

At least & at most once functionality is nice when you care about approximate results. But if you need exact results, you want
exactly once msg processing.

### Transactions
Take kafka input, process it, and output some msgs atomically to other kafka topics.

Internally, committing offsets for a given input topic is identical to writing to an additional output topic. Because the offsets
themselves are just topics named _consumer_offsets.

This means if we can solve the problem of **having atomic kafka writes** in general, we can solve the problem of exactly once processing,
because it means we can **both** produce an output msg(s) to output topic AND ALSO commit the offsets to consumer_offsets topic **atomically**,
meaning that both of those writes go through or neither of them do. Also no consumer can read those messages until they commit.

Note that kafka transactions are atomic inside kafka and don't provide guarantees for outside of kafka(like a db).

### What can't transactions do?
#### When dealing with external stuff
1. So if the consumer was also publishing to a db, that db is not in kafka. So it;'s possible that the consumer published to a db(through a kafka tx),
but it didn't commit the offset due to some problem, and then we're unable to rollback the data in db(but we can rollback the kafka msgs).
So dealing with external stuff is not a valid usecase for kafka transactions.

Review:

If we're publishing to an external store, this can't be done exactly once(the data in external store can't be rolled back as part of the
kafka tx)! This is because db ops are not part of the kafka tx.

If our processing logic consults an external store(or is not deterministic), retrying a msg may have a different result than it's first run!

#### atomic reads
Kafka transactions are not **atomic reads**! If we publish multiple msgs atomically as part of a kafka tx, it's not exactly easy for our
consumer to say: Oh wait, I know that all three of these msgs are linked as part of the same tx. Because the produced msgs inside of tx might
be produced to different topics and a consumer is not necessarily reading all those msgs at the same time. 
So it's not trivial to make that distinction.

So no easy way to fetch all msgs in a tx on the same consumer.

Transactions can solve these failure scenarios:
1. producer publishes the same msg multiple times because it didn't receive an ack from the broker(aka RPC failed)
2. consumer processes a msg but fails to commit the offset to the broker. In other words, consumer fails and can't commit input read offsets but publishes
an output msg corresponding to that offset. So we have the output msg but we didn't commit the offset. This results in multiple same output msgs.
3. producer fails temporarily, we bring up another, and then the other one comes back, leading to two producer publishing to the
output topic(zombie). To fix this, we can fence off the zombie using fencing tokens.

Note: In 3, producer is the one that actually consumes from input topics and writes the output to output topics, so it's a **dual** thing.
**So consumer is the one that consumes from input topics & writes to output topics.**

### Solve 1 (producer publishes a msg multiple times due to not getting ack) using idempotent publishing
Every time producer wants to publish exactly once msgs, the idea is to assign that a sequence num. So if for whatever reason the
broker stores the msgs but it's ack is not received by producer, producer would retry with the **same** batch seq number, the broker is
gonna reject these msgs since it has already seen that seq number before.

### Solve 3
- Each consumer is configured with an id(**tx id**. Yeah tx id not id, each consumer gets a tx id), that gets registered with kafka on startup.
- Also each tx itself gets a **epoch num** 

So let's say a producer is registered with tx id `x`. It goes to a component called transaction coordinator. Coordinator gives this tx
an epoch num of 1. But then the producer goes and another producer comes up instead. It also goes to tx coordinator. But this time,
coordinator gives it epoch num 2. It's also going to go to all topic partitions that are involved for this transaction and say:
since you're part of this tx(you receive msgs as a part of the tx), you should reject any msgs from producer with tx_id:x and epoch=1.

So now that output topic has a bit additional validation to say to the producer: hey I'm not gonna accept msgs from you in tx mode,
because you're a zombie producer.
![](img/kafka-transactions/3.png)

#### Caveat in problem 3
When using tx id & epoch nums, we could have this caveat:

We have producer with tx id: x & the transaction has epoch: 1. It writes to topic 1.

Now it goes down, so another producer is brought up, again with tx_id: x, but it   get's epoch: 2. But for whatever reason,
in this tx invocation, it only writes to topic 2 & 3. So topic 2 & 3 prevent msgs coming in from a producer with tx_id: x & epoch: 1.
Because the coordinator tell them to. But the topic 1 does not prevent this. Because the new producer didn't want to write to it.

So topic 1 doesn't prevent zombies to write to it.

Is this inherently problematic?

Yes, because maybe the zombie might write to topics 2 & 3, in addition to 1. In this case, the mgs for topic 1 will go through(to the broker),
but the msgs for topics 2 & 3 won't.

So ideally tx_id should be used on processes that are all reading from and writing to the **same** output topics.

![](img/kafka-transactions/4.png)

**There is one tx manager per broker.**

### Writing transactional data
So if I'm a consumer meaning getting data from an input topic and I wanna make an atomic write to multiple output topics, how do I 
perform this atomic write? This means all writes that are part of the tx, need to be committed or aborted. Also if they're not all committed
yet, they shouldn't be visible to downstream consumers(that are consuming from output topics).

Let's say producer wants to write to topics 1 & 2. It does:
1. initialize the tx to transaction manager. Since there are multiple transaction managers, we need to know 
which broker is responsible for the transaction manager associated with this producer’s `transactional.id`.
**The transaction manager lives on the broker that is the leader of the partition assigned to 
the producer’s `transactional.id`.**
2. The transaction manager (coordinator) writes a message indicating the start of the transaction to the 
appropriate partition in the internal `__transaction_state` topic.
3. producer writes data to topics 1 & 2.
4. Once producer publishes all the data, it asks the tx manager to commit. Tx manager adds a msg called start_commit to transaction log topic.
5. Now even if this tx manager goes down, a new tx manager comes up from one of the replicas. It sees that there's a start_commit and will continue
6. tx manager writes commit msgs again to the output topics. So in addition to the data written by producer, the tx manager
writes a msg indicating the commit(look at the img, there's data + commit in topics 1 & 2).
7. tx manager writes an end msg.

Note: If there's an issue, the tx manager writes an abort msg.

So we basically described a two-phase commit.

The most important thing is how consumers read the msgs inside of tx. The guarantee we're trying to hold with txs, is any data
can't be read until commit is written.

### Reading transactional data
If the consumer is in read_committed mode, it's only gonna see the msgs that are committed.

Waiting to see tx commit markers in a queue means that we don't do buffering on the consumer, thus reducing memory pressure.

So if consumer sees the `start` msg, it won't read anymore msgs until the corresponding `end` marker for that `start` msg comes in.
Now imagine there's a in-memory hashmap in consumer that has all the ongoing txs, just so that when an `end` msg is received,
the consumer knows which tx is resolved.

Q: Why if consumer encounters a start, put that new tx aside and still process the msgs until the end marker of that tx comes,
and then processes that tx?

A: 
1. That would put a lot of memory pressure on consumer. Since it has to buffer those tx msgs and if it doesn't have that much memory,
it would be a problem.
2. since we have to process everything in order in the log, we're buffering the tx msgs, but let's say the end marker never does.

So instead of doing consumer-side buffering, we do broker-side buffering. One of the beautiful things of kafka is that it doesn't matter
if the consumer gets behind. Since everything is persisted on disk. So we can let ourselves get behind and 
then catch up to broker some times later.

![](img/kafka-transactions/5.png)

### Transaction performance
Yeah it's 2-p commit, so technically it could be expensive if we ran txs all the time. But in reality, we prefer to run them in batch.
Because: Regardless of how much data is written to topics, we're still gonna be writing the same number of commit msgs to every single topic,
and still 1 commit to transaction log topic, we're still communicating with the tx manager the same amount of times.

So the idea is, if we write a lot of data in every tx, the overhead of actually performing the tx relative to the amount of data
written to each topic, becomes even lower. So as an optimization, kafka does a couple of things(core kafka behavior):
rather than sending as many msgs as there are topic partitions, it'll only send  as many msgs as there are unique brokers and then
that broker itself is gonna fan-out the msgs to all the topic partitions. Because the less frequent we run txs, the less it's gonna affect
the throughput of kafka. So if you’re writing more data per transaction, the fixed overhead becomes smaller relative to the data size.

In practice, for a producer producing 1KB records at maximum throughput, committing msgs every 100ms results in 
only a 3% degradation in throughput. This is in contrast to running a commit(instead of batching) every single time we actually wanna commit.

But the tradeoff is consumer has to wait longer before reading the committed msgs.

So by batching, txs don't make a huge hit on perf.

### Stateful consumers
We spoke about read-process-write, which is the consumer reads from input topics, runs a stateless func on input msg and then
publish the res to output topic. But then what if we wanted to maintain state based on the input msgs? We don't have any fault-tolerance
for this stateful consumer, if it goes down!

This is not true. Because the **state can be abstracted away into a topic**.

For better understanding: remember that a db is applying all entries of write-ahead log to a hashmap. 
In the same way, that's what cached state in memory is.

So if we represent our cached state in a kafka queue, the flow is like this:

Consumer reads msgs from input topics, processes them and then produce the output into the internal write-ahead log topic
and commits offsets of the read msgs.

So now instead of only having input-output kafka topics, we have 3 topics and we have transactions between every **pair** of these topics.

So between first two topics(input and internal write-ahead log) we have txs, so that the state is not lost.

Then between write-ahead log and output kafka we have another tx.