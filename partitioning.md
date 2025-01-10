**Note: Data locality means, when we do a query, all the required data is on one node and we don't need to make network calls
to multiple db nodes to get the result of the query.**

Note: Doing multiple network calls to multiple db nodes, makes latency larger and could fail.

### What is partitioning?
When our data doesn't fit in one node. We need to break it in order to fit it. Replication doesn't help us here, because
even if we do it, it wouldn't split the data. The entire data still needs to fit on one node.

We can't split the data randomly. Because when we wanna retrieve it, we need to know on which node it lives.

We can do it in multiple ways:

### range-based partitioning
For example partitioning based on first letter of name. So A-C goes to one partition 1, C-E to partition 2 and ... .
Ideally all the partitions are on different physical nodes. So we have to read and write to multiple nodes on the network
which is slow. It's always optimal to read and write to just one DB node. And we have this pro in this approach, since similar
keys are on the same node. And likely, our range queries only require one node.

- Pros:
    - similar keys will be on the same nodes, good locality for range queries
- Cons:
    - hot spot partition problem. Here, not many people have their first name starting with x and z, so that partition is under-utilized.
    But some other partitions like A-C would have a lot of data. It's a hotspot partition, in the sense that not only
    it has more people reading/writing to it but it's also gonna have more data and therefore queries to it are gonna be slower. So we might
    even have to further partition this partition.
    So it would be better if we shard in a more intelligent way, so that we don't have to re-shard a shard.

### hash-ranged-based partitioning
Let's say we have 4 nodes.

Instead of partitioning based on first letter of name, we have a hash func that takes a key and maybe spits out a number in range 0 - 1000.
So each of the nodes take 250 units, so like 0-250, 251-500 and ... . So a hashed key goes to the correct node.

If our hash func is chosen correctly, keys would distribute evenly and we won't have hotspot problem anymore.

But let's say in instagram and we wanna partition the comments based on their post_id. In a celebrity account, there would be a lot of
comments on one post. So all of them would be to one partition. Therefore, we have to re-shard those.

So even hash-ranged partitioning isn't perfect for avoiding hotspots if certain keys have a lot of data.

- Pros:
  - relatively even distribution of keys, less hotspots
- cons:
  - no data locality for range queries

To compensate data locality problem, we can use secondary indexes.

### secondary indexes
A secondary index is an additional index to our primary index.

Let's say data of nba players is partitioned by their name through hash-ranged partitioning. Now we want to partition by height.

So we have to store a second copy of data on each node. We store a copy of data on the same node in the order we want.

We're keeping a second copy of data on each partition sorted by height. Let's say the query is find people with `height = 6.3`.
Well, we have this useful secondary index on height on each node. But to find all results, we still have to read from 
every partition on different nodes. It's slow and has more points of failures. So this was the **con**.

Pro: When you make a write, you don't need to do extra work, because all the work for creating the data on 
secondary index for this new write is on one single node. This is in contrast with global secondary index.

### global secondary indexes
Instead of maintaining a second copy of data on each node, global secondary index takes the entire index of all of our data regardless
on which shard they're on, and shard that.

Let's create a global secondary index of all players: shard 1 keeps the global secondary index of all players that have height < 6.3.
Note that this data is stored in addition to the original data. It's still a copy.

Shard 2 has data with height > 6.3 .

- Pro: So now for querying height = 6.3, we only read from **one** node.
- Cons: In writes, the actual data could get hashed to shard 1, but it's secondary index is on shard 2. So we have to write to multiple shards
or nodes. In other words, we have to write to one node for actual data (primary index) and a second node for updating secondary index, which slows
down writes and requires **distributed transaction**.