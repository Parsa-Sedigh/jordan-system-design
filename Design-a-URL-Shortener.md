availability > strong consistency

## Supported scale
- URL creations: 600 M / month = 228 / second (writes)
- URL retrievals: 10 B / month = 3508 / second (reads)

How many characters should we use for our URL suffix?

While we can configure some expiration dates on short urls to make capacity for new links, let's see how this goes without any expiration.

600M creations / mon = 7.2 B / y = 72 B / in 10 years = 720 B / century

So we need to support 72B unique suffixes.

NOTE: There are 26 english alphabets. 0-9 are also 10 chars. We wanna have our short url to support both of these, so
we can use 36chars.

If we use 8 chars for suffix, we have 36^8 = 3 trillion possibilities. Note that for each char we have 36 choices.

Since each char is english chars and numbers, it takes 1 byte.

## URL mapping table
schema:
- short url -> 8 bytes
- long url -> 50 bytes
- creator_id -> 8 bytes
- expiry_time -> 8 bytes

74 bytes per row.

## Number of database nodes
### reads
3805 db reads / second. Simple reads, can probably be handled on a single node.

### writes
3508 writes / second

Can most likely be handled on a single node, depending on write complexity.

### Disk
74 bytes * 720 B = 70 * 70 * 10^9 = 5000 * 10^9 = 5 * 10^12 = 5TB

Can be handled on a beefy node, but likely worth horizontally scaling.

## High level design
Caching enables us to avoid hotspots across db servers

NOTE: We use look-aside strategy with LRU eviction for cache.

Since we want the short urls to be distributed across many db nodes, we can do so by assigning each db node a contiguous range of
the short url space.

We can seed these short urls by creating a row for all possible short urls and an initial expiration of 0, so that any new attempts to
create a short url will succeed.

Look for the first row that we're passed it's expiration time(expiry_time < now()) and claim it for the user.

## Retreving a short url
How do we know which db node to find the url that user wants to redirect?

We can keep a static mapping of key ranges in db nodes, cached on our app servers. We don't anticipate this mapping to change a lot.
But we will talk about what to do if we want to make our servers more adaptable.

- Postgres uses single leader asynchronous replication.
- All writes go to leader.
- Replicas make data durable and serve read reqs so they increase the read throughput
- in case the leader fails, postgres will promote one of the replicas to take over(fail over to replica).
- Since the leader is not waiting for acks from replicas before considering a write as complete, it's possible that a leader fails before
all of it's data is durably replicated. In this case, we may lose some data when a replica is promoted to become the new leader.
But this is a url shortner service and data durability is not that much important as in financial cases.

To find a pre-defined row in shortUrl table named url_mappings:
```sql
with candidate as (select *
                   from url_mappings
                   where expiry_time < now()
  limit 1
  for update skip locked; -- if the row was locked, skip it and find the next one, since we don't care which short url row we claim, this is fine
)
update url_mappings
set expiry_time = now() + interval '1 year',
    user_id = 4
    long_url = '...'
from candidate
where url_mappings.id = candidate.id
returning url_mappings.*;
```

Hash(short url) % num_partitions: to figure out which partition a short url belongs

## Deep dives
- db sharding
- citus
- hashing vs key generation service
- choosing a db
  - index
  - replication
- caching

### key rebalancing
Currently, to claim a node, we're manually performing random request routing from app servers to db nodes. Instead, we can use a 
db that manages partitioning and routing automatically.

In the case where we needed to **rebalance** the key ranges that each db node holds, things can get complex, because we need to
notify the app servers of rebalancing. There may be a period of time where app servers reach to the wrong db node.

Cockroach DB performs req routing to shards and automatically rebalances the partitions.

Single leader asynchronous replication has risk of losing data on failover.

## supporting multiple regions
We could place a subset of pg shard leaders in each region. Our app servers would only route url creation and get to their own region.
Cross region reads could be sped up by using read-only replicas 