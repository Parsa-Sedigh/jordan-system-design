https://www.youtube.com/watch?v=_5mu7lZz5X4&t=93s&ab_channel=Jordanhasnolife

## Stream processing
### stream processing review
We have 2 sets of nodes in our system: producers and consumers. Producers send events to consumers, which go through a broker.
Producers and consumers communicate with one another via events. Now we **could** send those events directly(in some ways, the stream
processing do this, we can send them directly over TCP or UDP or sth like zeroMQ which works on top of those), **but** the majority of time,
what we wanna be using is sth called a broker. Because it handles the load of dealing with all of those connections for us and we can
scale it independently of our producer and consumer nodes.

2 main types of brokers(different in arch):
1. in-memory message brokers: some implementations are rabbitmq, activemq, amazon SQS
2. log-based message brokers: some implementations are kafka, amazon kinesis

## in-memory message broker
All of the events are kept in memory. We can put them in memory as a linked list or an array, but the general gist is we want to be
representing a queue. So we have this queue of messages and every single message gets sent to a consumer and eventually once it gets
sent to the consumer, it's going to be deleted from the equeue(at least temporarily) and then once the acknowledgment comes back from
the consumer, saying this message was handled, then that message **really** does get deleted from the queue.

Q: Let's say we have this sequence of messages: C -> B -> A . Does it mean that if B is before C in the queue, B is gonna get processed
before C gets processed?

A: This is not necessary true. Because when B gets sent to a consumer, for example consumer 1, message C can instantly be sent to consumer 2
as long as consumer 2 is ready for it. So consumer 2 could process message C faster. Therefore, it is not the case that we necessarily need
to wait for the completion of processing of the prior message before sending the subsequent message to a different consumer.
We're effectively using round-robin in order to ensure that we can get the maximum possible throughput in this situation.

So if the network connection is slow(for pulling or pushing the message by/into a consumer) or the message is hard to process, maybe it's a
heavy task for the consumer, it's possible that a message that is further back in the queue(added later) gets completed before an earlier one does.

General conclusion of in-memory message broker:

We're using round-robin to maximize the throughput but it can lead to out of order processing. The only way to ensure getting **in-order** processing,
is by doing **fan-out** where instead of having multiple consumers for one queue, we would have multiple queues(or multiple partitions instead of queues)
and then we would have one consumer exclusively read from one queue(partition) and other consumer exclusively read from the other partition or queue.
But this kinda defeats the point! Because we wanna maximize our throughput.

Note: The other thing is: the messages in memory are going to be deleted. When they are processed successfully, we get rid of them(erase them from memory or
garbage collect them). What this implies is we have **poor fault tolerance**. If this machine were to go down, unless we have sth like 
a write-a-head log to write all of our events to disk, we're just gonna lose all of these events and additionally of course we can't replay these
messages because we're deleting them once they are consumed.

OFC, instead of using a write-ahead log, which is gonna slow our writes a bit, because writes to memory are always faster than writes to disk,
what we could instead just do is effectively using a write-ahead log which is a log-based message broker.

## log-based message broker
The messages are stored on disk. In this broker, we have sequential writes to disk instead of random because sequential writes on disk are
faster than random ones.

So the messages are next to each other on disk.

What a log-based message broker does, is make sure all these messages are held in the exact order in which they came in. It also ensures
that all consumers read the messages within a single queue in the same order in which the messages are stored. To do this, it stores the
location of the last message that each consumer has read. It does this after the consumer acked the message.

So there is no round-robin to be done here, in fact if we wanted to increase our throughput for our consumers, what we would have to do, is to
create a second broker and we assign some consumers for one broker and others for another.

Note: Because we have to read every single message in the queue **in order**, that does technically lower our throughput. The reason being
let's say message 3 takes a lot of time to process, let's say consumer B starts processing it. It means that it's gonna an additional 60 seconds
before all of the remaining messages in the queue are processed. So any slow message is gonna be a bottleneck for all the others
So it slows the processing of the rest. To fix this, we would have to partition our queue with additional consumers reading from other
partitions in order to increase the throughput.

Nice thing about log-based message brokers:

The messages are durable. Not only they are on disk, if our machine were to go down, but more importantly we don't delete them once we've
read(process) them and it means if for whatever reason we want to add additional consumer in the future to reread those messages, or perhaps
we're just convinced that we missed a message we can just re-read them all and perform all of our processing again.

Recap: All consumers reading from a queue, get every message in order. One slow to process message, slows processing of the rest. We need to partition
the queue to increase throughput.

More durability -> since messages are on disk, can be replayed later.

## conclusion
Each type of message queue performs better in specific use cases:
- in-memory: 
    - If we want maximum throughput. Because it's fast to read and write to memory.
    - we don't have single messages in queue being a bottleneck. We're delivering them in round-robin and as a result of this,
      we can ensure that every single message is going to be processed as soon as there's a consumer that's available to process it
    - the messages are gonna be processed out of order
    - the messages are not durable. So we can't replay them
    - Ex: User posting videos to youtube. We know every single time a user uploads a video to yt, yt has to encode it on it's own backend.
      It doesn't really matter to youtube whether my video gets encoded first or another user video gets encoded first or other user's video gets
      encoded first, the work just has to get done and then it can be stored in our DB. So because the order doesn't matter, because they're
      not gonna have to redo the encoding work in the future and they just wanna encode as many videos as possible to decrease the latency between
      when we post the vid and when our video is available, then the in-memory broker is really great for them.
    - EX: A similar type of situation would be sth like twitter. In twitter, users typically tend to make a bunch of posts and those posts are
  then delivered to all of their followers dedicated cache which will represent their newsfeed. Now OFC if I post half a second before you,
  it doesn't really matter whether my message gets into the cache before yours does. So in this case also in-memory broker is totally fine.
- log-based: Want all items in queue to be handled by one consumer, in order and also has the ability to replay.
    - EX: sensor metrics: Because the orders of these matter. Let's say we wanna calculate a trailing average of the last twenty seconds. If 
  we were start to processing these events out of order, the result is wrong.
    - EX: Each write from a DB that we will put in a search index(this is called **change data capture - CDC**). Because the new writes overwrite the
      previous writes. In this situation, the ordering of the writes of the DB should be preserved. Because let's say x = 5 and
      then we say x = 7, well the result of this(7) is very different than x= 7 and then x = 5 which would overwrite the 7 and the result would be 5.
      Also if you want to add new derived data sources down the line, it would be great if those events didn't get deleted the second
      we put them in our search index. Hence a log-based message queue is appropriate here.

