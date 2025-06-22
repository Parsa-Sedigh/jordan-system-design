### Replication introduction
1. each topic (and each partition of it) are replicated. So every partition of each topic, is going to
      be replicated a certain number of times(on certain number of brokers).
2. each write goes to the leader replica and each read comes from the leader. So by adding more replicas, you're **not** going to be
getting more read and write throughput. Because everything is going to and from that single leader. Another consequence is that makes 
data consistency simple to reason about. We can almost say it has strong consistency.
3. when a producer writes a msg, we configure how many of the **"in-sync"**(different than normal replicas) replicas need to 
acknowledge it for the producer to consider it successful(otherwise we retry). For a producer, we have multiple opts for dealing with acks:
  - no ack needed: acks = 0
  - just an ack from the leader: acks = 1
  - all "in-sync" replicas must get the message and ack it(producer in addition to an ack from the leader, waits for the leader to
  get ack from all in-sync replicas): acks = all
4. a msg that is on the leader broker, is not considered "committed" (readable or visible to consumers) until it has been
acknowledged by a **all** of "in-sync" replicas
  - as long as one of these replicas is alive, msgs are durable

> `min.insync.replicas`: The minimum number of in-sync replicas that must acknowledge a write for it to be considered **successful** 
when the producer uses acks=all.
> - It does not directly control what “**committed**” means internally to Kafka — that’s always controlled by “replicated to all ISR members”.
> - But it ensures that a message is only acknowledged back to the producer when it’s been written to at least N in-sync replicas, where N is min.insync.replicas.

> In Kafka, a message is considered **committed** when:
> - It has been successfully replicated to all in-sync replicas (i.e., the entire ISR set),
> - Regardless of what min.insync.replicas is set to.

About 4: So you can set up a producer to consider a msg successfully produced when only the leader acks it(acks=1), but for a consumer to
**see** a msg, the msg has to be replicated to **all** in-sync replicas(only the leader is not enough).

Since the number of in-sync replica set can grow or shrink, there's `min-insync.replicas` setting on producer and with this, we can say:
if there's only 1 in-sync replica running, by setting this setting to num higher than 1, we say: msgs are not written to any broker,
even though they were acked by that one running in-sync replica.
So consumers of that topic can't see those msgs yet until more in-sync replicas ack that msg.

Q: Why in-sync replica set can grow and shrink?

A: It can shrink when an in-sync replica set is slow because that's gonna stop us from processing any new msgs if current num of **available**
in-sync replicas gets smaller than `min.insync.replicas`.

> A message is considered committed when:
> It has been replicated to the leader replica and to all of in-sync replicas.

### In-sync replicas
- each partition maintains a list of "in-sync" replicas in zookeeper.
- Messages must be written to the leader + replicated to all in-sync replicas (ISRs) in order to be considered committed. This means we're bottle necked by the
slowest of the in-sync replicas. For example, one of them could have network issue or hardware or maybe it has a lot of load on it,
issue and is unable to process msgs. This can be really bad. However, we can kick in-sync replicas from the set. 
Also note that, in-sync replicas can return to the in-sync set when they catch back up!
- if the leader of a partition fails, what we can fail over(fall over) to, has to be an in-sync replica.
You can tell this by the name of **in-sync** replica. Every single in-sync replica is completely up to date with the leader partition.
You might say: "when we write a msg to the leader, it may have a few more msgs than some of the in-sync replicas. So they're not in-sync, right?"
No, this is wrong. Because that msg is not considered committed, no downstream consumers can read it from the leader. So effectively,
any in-sync replica is just as up-to-date as the leader.

> If one or more in-sync replicas are slow and the number of acknowledgments falls below `min.insync.replicas`(assuming acks=all),
> new messages will not be written at all, and consumers configured to read only committed messages (e.g., transactional consumers) may not see them.
> This behavior protects against data loss in case the leader fails — because Kafka cannot guarantee durability if too few replicas are in sync.
> Kafka refuses to accept the message rather than risking an acknowledged write that could be lost on a failover.

Note that in-sync replica set is dynamic. We can kick replicas from the "in sync" set, when:
- they lose connection to the controller
    - controller is single component that makes replication and failover decisions in kafka. It's heartbeating all the brokers and therefore replicas.
    So if a broker is down, it's replicas are down. So it will drop it from the in-sync-replica set which those replicas are part of.
    It's communicating with zookeeper as well. It uses zookeeper for leader election and storing replication info
- they fall too far behind the newly sent msgs to the leader
    - this threshold is configurable in milliseconds

Note: Zookeeper internally is using consensus to ensure that any writes made to it, are linearizable and they don't get lost.

Note: We want as many in-sync replicas as possible, because the more in-sync-replicas that we have, the more durable our msgs are.
In other words, we can tolerate more failures when we have more in-sync-replicas.

> So there are 2 phases when writing msgs:

| phase                     | requirement                              | effect                                     |
|---------------------------|------------------------------------------|--------------------------------------------|
| Acknowledgment (Producer) | Written to ≥ `min.insync.replicas` ISRs  | Message is acknowledged to producer        |
| Commitment (Cluster)      | Replicated to all current ISRs           | Message is committed; safe for consumption |


### kafka replication visualized
As you can see the msg d is on leader but not on ISRs. But we still consider those replicas as in-sync, it's just that `d` is not yet considered committed.

The only msg that is considered committed, is `a`. Because both followers that are in the in-sync-replica set have it.

If for whatever reason the bottom replica were to fall out of the in-sync-replica set, b and c msgs are considered committed.

In kafka, followers pull the leader in the same way that a consumer might pull from broker.

**High watermark:** lowest offset that **all** ISRs have replicated — not just replicated to #`min.insync.replicas`.
It allows us to tell which msgs are committed. So any msgs before the high watermark are considered committed and anything after, 
is considered not committed.

In img, the min of F1(follower1) and F2 is high water mark which currently is msg `c`.

Note: If for whatever reason follower 2 went down or it was following very slowly and it never got msg d, so that we couldn't consider `d` as
committed, eventually we would remove follower 2 from ISR set and proceed as normal. This way d would be considered committed because the ISR
that didn't have this msg, is now removed from the ISR set.

### Kafka leader failover visualized
When we failover to a new leader, it's gonna get an incremented epoch num.

![](img/kafka-replication/1.png)

#### Stale high watermark
Since a follower is async, it's conception of high watermark is a bit behind.

The reason is: When the follower reqs new msgs to get replicated to it from the leader, the leader is sending it's high watermark.

Now let's say leader is gonna go down before sending it's high watermark to this follower. So if the follower was behind when the leader
went down, the follower won't know the high watermark.

So it has a stale high watermark.

Note: Whenever we elect a new leader, we bump the epoch number by 1. So epoch number acts as a fencing token.

Leader epochs prevent stale leaders from writing.

When a broker is elected leader, it receives a monotonically increasing epoch. If the prior leader recovers later, it holds an old epoch.
Kafka uses this epoch as a fencing token: all writes to an outdated leader with lower epoch are immediately rejected. 
This guarantees that only the current, valid leader can accept writes and protects data integrity.

### Kafka replication optimizations
Kafka does not require that each replica flush it's log msgs to disk. Keeping msgs buffered in memory leads to big perf improvements

---

Another configuration that you can do: If all brokers in the ISR go down, that partition becomes unavailable and 
no new data can be committed. However, previously committed data is not lost if the brokers recover with their logs intact.

For this situation, kafka supports two operational modes, reflecting a trade-off between availability and consistency:
1. Safe mode (default): The partition will only recover when an ISR broker comes back online—ensuring no data loss,
though it may remain unavailable longer. This is the behavior when `unclean.leader.election.enable=false` (the default in modern Kafka versions).  ￼ ￼
2. Unclean leader election: If any broker returns — even one not in the ISR—the partition recovers faster, but at the risk of data loss.
This occurs when `unclean.leader.election.enable=true`, allowing non-ISR replicas to become leader and potentially miss committed messages.

If we pick the first option, we get consistency(because an ISR replica which is in-sync with the down leader, is chosen as the new leader).
But if we chose option 2, we get availability and might lose data, because we could elect a non-in-sync-replica as the new leader.

---

About 1: Typically it is assumed in a replication protocol that every single msg that we write to every node is gonna be written to disk
because it needs to be durable, so even after restart, we still have them. But in kafka, they don't do that because it's a huge perf killer.
They'd rather buffer msgs. This decreases the durability. Also note that in reality there are a lot of disk failures and so
actually having data on disk is not as durable as it seems. In other words:

> Kafka sacrifices immediate local durability by default to achieve massive throughput, buffering writes in memory and 
deferring disk flushes.

Q: Does this mean kafka is not durable after acking the producer?

A: Kafka does not fsync to disk before acknowledging writes. It simply writes to the OS page cache, which typically occurs quickly.
Therefore, if a broker crashes (e.g., via power loss) before the OS flushes to disk, even an acknowledged message can be lost.

Bottom line: Kafka acks based on in-memory writes, not on disk persistence.

Durability through replication:

Kafka’s safety mechanism isn’t local disk flushes—it’s replication across in-sync replicas (ISRs). 
With `acks=all` (or -1) and `min.insync.replicas > 1`, Kafka waits for multiple brokers to receive the message.
Even if one node crashes before flushing, others may still have changed their page cache and can recover the data

In rare cases, “committed” messages can still be lost:

Although unlikely, data loss is possible if all in-sync replicas crash simultaneously before flushing to disk. 
For example, a power failure across the cluster before page cache flushes could wipe acknowledged data .

With proper replication (`acks=all` + `min.insync.replicas`) and multi-zone deployment, such events are extremely unlikely.

---

On startup, Kafka attempts to balance partition leadership across brokers.

- Each partition’s preferred leader is the replica listed first in its replica list.
- Kafka periodically checks and, when needed, moves partition leadership back to the preferred replica — this behavior is
  governed by `auto.leader.rebalance.enable=true` (default) and `leader.imbalance.per.broker.percentage` thresholds.

> Note: We want the leader partition of every topic to be equally spread out across the nodes in our cluster. 
Kafka controller balances the leader replica(partition) to be across different brokers.
For example, if we have 4 different nodes and maybe 16 topics, each node will be the leader of 4 topics. So we can spread the leaders
in round-robin way.

Each topic is assigned one particular node at startup time and we call that node the preferred node. If the preferred node goes down,
the leader partition will be chosen from another node. But now the problem is there's more load on that another node(less load balancing).

Now if the preferred node comes back up and that other node goes down, we move the leader partition to the preferred node, not another node.

### Kafka replication vs quorum
The distinction of quorom based replication vs kafka replication approach(in-sync replica approach)

quorum based approach:

The replicas that need to acknowledge a given write for it to be considered committed, needs to also overlap with the set of replicas
that would be able to vote for potential candidate for becoming the leader(in case of fail-over). With this, we know that
one of those replicas is always gonna have all of the committed writes on it. In other words, this ensures we don't lose any data.

- a **majority** of nodes must have the msg for it to be considered committed
- on failover, a majority of nodes must vote for next leader to avoid data loss

in-sync-replica approach:
- **all** in-sync replicas must have the msg for it to be committed
- on failover, choose **any** of the in-sync replicas

By using quorum, if a node is slow or goes down, we're not bottlenecked by the slow nodes, because we just need the majority.
On the other hand, in ISR approach, we're bottlenecked by the slow nodes. But since ISR approach is dynamic(since we can kick out
the slow nodes), in reality, it's not that bad.

Another advantage of ISR approach is we can handle more failures. Because let's say we're using quorum with 5 nodes. If 3 of them
go down, we can't proceed anymore. But if we had 5 in-sync replicas, we can tolerate 4 failures. So we just need 1 alive to produce and
read from it.

So ISR approach can tolerate more failures.

Latency in quorom systems are not bottlenecked by the slowest nodes, but if a majority of nodes go down, they can't proceed.
In-sync replica sets are dynamic, evicting the slowest nodes and can tolerate all but one in-sync replica failing.

### Conclusion
By tuning kafka parameters, we can achieve full replication of msgs! In practice, it's very unlikely that all replicas go down.

At the end of the day, kafka doesn't provide strong consistency since all the replicas can go down, but it's pretty close.

Since acknowledgements come in batches, it's actually more reasonable to wait for responses from **all** ISRs, per msg. Kafka benefits significantly
by using batching to greatly improve it's throughput while providing strong consistency guarantees.