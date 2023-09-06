## Two phase commit
We partition our DB when our dataset becomes too big to be stored on one individual node and that means now we often find ourselves
having to write to multiple physical computers at one time for a variety of reasons:
1. cross partition writes: sometimes you make a write and it needs to go to multiple different places
2. global secondary indexes: Instead of just having a local secondary index where we index all of our data per node, we take all of the
aggregate data across partitions and we split up that global index such that a piece of the index is on every single node

We need atomicity here, because we want to write to multiple places like node 1 and node 2 and they represent different partitions, now if
write to node 1 is successful but write to node 2 is not, our DB is now going to be in an inconsistent state and we're going to probably
have some incorrect reads which generally we don't want. Eventual consistency is OK because we know it will resolve itself but incorrect
reads will not, those are going to be persisted forever.

## How does it work
Let's see the phase where it doesn't work(when we can't go through with two phase commit aka there is an error):

First we have a node named coordinator node and even though we're calling it a node, what this might end up being is just 
an application server. We have a client(that dummy on top) who wants to perform a write and the client is reaching out to our
application server(coordinator node) and the app server is sending two different writes to our two DB partitions.

The first thing the coordinator node is going to do is say: Alright, I'm trying to start out a two-phase commit write because I want to
write to two different partitions. So the first thing it does, is it's going to say: I'm going to reach out to node 1 and say:
I'm sending you message 1, are you ready for it? Node #1 is going to look at itself and say: hmm, can I actually go a head and commit
this thing? I'm gonna run a local transaction, so here's my write a head log, can I write it down? Is it gonna cause an error? and if
everything is good, it's gonna grab the locks for that row, so that nothing else can modify it and it's going to respond back to
our application server with an ok. Now in the meantime from when node #1 responds ok, until it hears back from the coordinator node,
it's just waiting and the locks are still grabbed. 

The coordinator node at the same time in parallel, is going to reach out to node #2, with another message saying: Hey, are you ready to
possibly commit message 2? Then node 2 is going to look locally, it's gonna see the message 2 and it might say: hmm, I'm not sure I can
commit this(either it's gonna conflict with some other write that I have or perhaps I don't have enough space for it) and then it
respond back to the coordinator node saying: no. Now what the coordinator node has to do, is going to send a message to node 1 and say:
abort and it's only then that node 1 can unlock the locks it grabbed.
![](img/two-phase-commit-distributed-transactions/1.png)

We can go through with two phase commit:
It's the same with the previous case, but now the node 2 sends back an OK. Now what the coordinator node is going to do is the following:
In it's own local commit log, it's gonna say: Hey, we have a transaction that I know I want to be able to commit, with message 1 and
message 2 and from now on, if I happen to go down(coordinator node goes down), it can read back from the commit log and it can recommit it.
So even if our coordinator node goes down, keep in mind that the node 1 and node 2 are both waiting to hear back from the coordinator node
and while they're waiting, they've maintained grabbing the locks for all the rows that are going to be part of this commit. Then let's say
the coordinator node is ready to finally send the commit and now it's gonna send the commit command to node 1 and 2 and then now node 1 and 2
can commit that local transaction. So they're gonna put that in their write ahead log, commit it and now they can unlock their local locks and
then once they do that, they can respond back to the coordinator node, saying: ok(we're done).

So all the coordinator node is doing is ensuring that all of the transactions either go through or all of the transactions do not go through.
It's important that they are **atomic**.
![](img/two-phase-commit-distributed-transactions/2.png)

**Commit point**: The first time the receiver nodes respond OK and now they're holding their locks and while those locks are grabbed, no other
transactions can go through on those same rows.

### Problems with 2 phase commit(2PC):
Too many points of failure:
- coordinator goes down? No transactions can proceed, receiving nodes hold locks and can't touch rows
- receiver goes down? transaction can't commit, coordinator needs to send it's messages forever until it comes back up

Note: We have a single coordinator node and if it goes down, nothing is getting committed.

Note: If any of the received nodes go down, the distributed transaction can't commit because the coordinator node can't get an OK from all of
the receiver nodes after the commit point(after commit point means the receiver node said: Hey, I'm good to commit this previously, but now
it's down).

We've basically said: If we're past the commit point, we need to get this distributed transaction done no matter what it takes, we're gonna try it
forever until we can get it done.

So we have very little fault tolerance here.

## Conclusion
- Distributed transactions are hard. We'll see how to make them more fault tolerant, but they're still slow.
- Where possible, be smart about how you partition your data to avoid them.

Note: distributed transactions are a necessity for things like: cross partition writes and global secondary indexes, these are very dangerous. Because
we run the risk of having a failure and when we have a failure, our system is unable to proceed. Thus things like cross partition writes and global secondary indexes
in distributed systems are generally things that we want to avoid when possible. There are definitely other ways of being smarter about
how you shard your data or possibly even improving your data locality with sth like a noSQL(like a document store DB) that are going to
allow us to generally be able to avoid these types of cross partition writes.

Note: While 2PC can be improved(we can do that with distributed consensus), it's sth that generally you want to avoid.