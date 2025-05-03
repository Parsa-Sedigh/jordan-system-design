# Distributed Locking Design Deep Dive | Systems Design Interview Question 24

## Functional requirements
Is more or less the same thing as a typical lock on a single computer system. The main difference is multiple different nodes in
the cluster can be accessing it and locking or releasing that lock.

Why we use them?

There are a couple of reasons we tend to use them:
1. **protecting shared resource** like file modification situation: imagine we have a S3 and all different nodes in 
the cluster wanna be able to edit some files in s3 and they wanna be able to edit the same files. 
If two of them are editing one file at the same time, that file is gonna get corrupted.
So they'll grab a distributed lock to say: Hey, I'm gonna be editing the file now. That way no other nodes in the cluster
can do the same.
2. **preventing duplicate work**. The nodes in the cluster are pulling from a queue and processing them
and then uploading them to s3. Here, we don't want to do duplicate work. So every time a node wants to process a chunk of vid,
it grabs a lock corresponding to that chunk and then that way every other node in the cluster will know they're not able to
grab that distributed lock, therefore no duplicate work(double work) will be done and we won't waste any compute.

## capacity estimates
Disk and memory requirements are not too important for sth like a distributed lock. But the number of machines that may be trying to grab
the lock is important. Let's say we can have up to 1000 nodes in our cluster trying to grab this lock at once.

## API design
- grab_lock()
- release_lock(fencing_token): we use fencing_token to be sure it's the right machine doing the releasing
- do_external_operation(fencing_token)

## Database schema
For now, just who's holding the lock, maybe we add stuff later.

## Architectural overview
We wanna have sth that allows for exclusivity. Only one machine can be holding this lock at a time.

In the face of fault tolerance, whether it is the locking server that's down or the machine holding the lock that's down,
things have to still keep working.

### If the node holding the lock is down
Now it can't release the lock. How the system is gonna progress? How anyone else ever going to be able to grab it?
We have a couple of options for this:
1. we have a lock server that's keeping track of everything. The lock server can put a time to live on the lock and say after 5m,
the lock server releases it automatically. This time to live depends on type of work and the expected amount of time that you think
it's gonna take for the process to end to configure that ttl
2. periodically have your machine send a heartbeat to locking service and if the locking service stops receiving those heartbeats then it
can assume that machine is dead and release the lock. This should ring an alarm in your head. Because you can say: "we can't know for certain that
the machine that's currently holding the lock is actually done with it, it may still be doing work" and that's correct. It may still be.
Maybe it had a process pause or garbage collection, it may just be taking longer than normal for whatever reason or maybe there's a network delay
or issue when sending heartbeat or unlocking the lock req. But the point is: this machine might still be thinking it's holding the lock because
it never actually called `release_lock()` on it's own. So then it might go to s3 and try to edit a file and then boom, there's corruption because
another node already grabbed the lock and now they're editing at the same time. TODO: I don't think we would have a corruption here

How can we fix this discrepancy?

This is where fencing tokens come in. Fencing tokens are some sort of monotonically increasing sequence numbers that s3 is gonna keep track of
all the fencing tokens it's seen when machines edit the files that are on it. With them, a service can keep track of what's the last one it's seen
and only accepts greater numbers than that.

For example: A node had the lock, it got TTLed which means the lock server released the lock of that process(but the process still doesn't
know about it maybe it was down) and another process grabbed it. So let's say original fencing token was 34 for the one that expired and then
the new machine that grabbed the lock after it expired, got 35. The new machine starts editing files on s3.

Now the previous node comes back and wants to edit the files that is being edited by the new machine, but s3 is gonna say: I've seen the
fencing token 35, so s3 disregard the req with fencing token 34.

Q: How do we assign fencing tokens? What do we use for them?

A: We definitely don't want to use timestamps for fencing tokens. Because timestamps in a distributed system are unreliable. Sure you can
have that **one** centralized lock server assign timestamps, but we can't just have one centralized lock server because the entire point
is we're building a fault-tolerant system, so if we're only relying on one single server to be in charge of all distributed locking and
keep track of the state of who has the lock, what happens if that machine goes down?

Then another one has to come up and because these timestamps aren't reliable, the new one might start giving out timestamps that are
actually lower than the timestamps given out by the first lock server, even though we know that this is after that.
So now we're gonna have another issue with fencing tokens.

Another option for using as fencing tokens is: what if we used a system like dynamoDB which has quorum reads and writes.

A quorum is when you have an odd number of servers, like DB servers, let's say 3 and to achieve a quorum you have to 
write to `two-thirds` of the servers you also have to be reading from `two-thirds` of servers.

When we're writing in dynamodb, the write gets a version number which is our fencing token. The reason sth like this works is
in quorums in particular, you know that by performing a quorum write and a quorum read, which means two-third of db servers are written 
and read from, we know the most update-to-date val is always going to on at least one of the nodes. So you can kinda ensure 
strong consistency. But the truth is we really can't! Because for example:

TODO: I think this paragraph has issues:

someone tries to grab a lock and it's only able to write to one of the 3 nodes and then another machine is gonna read from the 3 nodes
and say: if the lock is available or not and then it reads from 2 of the 3 db servers and actually one of the two nodes that it reads from,
is the one where the write from before was successfully written to. So now one out of two nodes is gonna set the lock is already grabbed.
So even though no one is actually successfully grabbed the lock, it's gonna look to the other server that the lock is grabbed and now the
other server can't progress. So quorums can't be used here.

---

We want to use consensus on raft.

Raft is fault tolerant, highly available consensus algo. It allows us to build a distributed log. Think of distributed log as a list of
writes where each write is an event and on top of that log, we're building a key-value store. The logs are gonna be replicated in a
fault-tolerant matter over the other raft systems. This is how it works:

Raft is gonna use an odd number of nodes and then one of those nodes is gonna be designated as the leader. Every single time a write is arrived,
it is first sent to the leader, the leader is gonna propose it to all of the follower nodes as long as a majority of follower noes are
able to say: hey I'm cool with this write, I can accept this. They're gonna send this to the leader and if the leader sees that the majority
of the followers have accepted that, it's gonna commit that write locally and then tell all the follower nodes to commit that write.
This way, it's never possible 