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
consumer to say: Oh wait, I know that all three of these msgs are linked as part of the same tx. Because a consumer is not necessarily
reading all those msgs at the same time. So it's not trivial to make that distinction.

So no easy way to fetch all msgs in a tx on the same consumer.