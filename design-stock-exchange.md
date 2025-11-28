### Functional requirements
1. create an exchange to match buyers and sellers of stocks!
2. provide an ordered list of ops that everyone can agree on (eventually)! Note: If server goes down, the new server should have the prev
orders in 
3. in addition to providing total ordering over all ops, we also want to provide an ordered list of activities related to 
**each client** (placed orders, cancelled orders, trades) back to each client. So for example, a trading company as our client,
wants to know all of their orders and trades in the correct order.

Note: Trades are executed when best bid >= best ask

### Matching engine performance
The matching engine needs to be fast!
- handles all incoming orders and cancellations, publishes data
- we don't want to involve any additional I/O(disk and network calls) here, everything run locally in memory.
  This doesn't mean that it's all going to be single threaded. But since if we make it multi-threaded, locking should get involved, but ideally
  we wanna avoid locking as much as possible, but there are some situations where we can't avoid locking.

Note: The matching engine is deterministic -> Meaning if it currently has state s, applying msg m to it will always produce the same state s2.

### Matching engine networking
We got this one box which is our matching engine and we have a bunch of parties who really care about what's going on in that matching engine.

The matching engine has many interested parties in what it is publishing:
- clients sending orders - vendors trading with the exchange on behalf of the customers. Those vendors are for example: vanguard, robinhood and ...
- banks/ companies - who care about verifying things look good
- other market data vendors - like bloomberg who get this data, modify it and sending it out to people for money

All of these parties are gonna be connected to the matching engine through proxy servers. Those proxy servers need to be getting the data
delivered to them fairly. Because if the data goes to one of the proxy servers well before others, they have more info than others and can make
more money trading. For this, we use UDP multicast. 


- tcp requires a bunch of one to one connections. Now, if we wanna send data to all these conns at the same time, sure we can have multiple threads to send
msgs out at similar times, but they're not gonna send msgs at **exactly** the same time. Whereas multicast is literally one msg that is sent out
at the same time to all these parties(over private network OFC).
- instead, exchange uses UDP multicast. Meaning losing flow control and reliable or ordered broadcast 

So we need to somehow mitigate the downsides of using UDP here

### Dealing with UDP
Since we're using UDP, msgs can get dropped or come out of order.
- to deal with msgs being out of order, we need flow control. Consumers of msgs should be fast.
- to deal with msgs being dropped, we use `re-transmitters`. Let's say matching engine sends msgs with sequences 4 & 5. But for some reason,
a client doesn't receive #4. It just gets #5. So it's gonna say: It says: I saw everything till #3 but now I got #5, which implies #4 exists.
Now even though the client didn't get #4, we got a few re-transmitter nodes for caching and storing every single msg. So client can ask
re-transmitter for the msg.

So re-transmitters allow us to deliver all msgs to clients.

Q: Is it possible that somehow all the msgs dont get to re-transmitter?

A: It's possible but we can mitigate it by having a lot of re-transmitters.

### Matching engine fault tolerance
The matching engine runs completely in memory, and can go
down(so even if it comes back up, we still don't have the state) -> We need multiple backups! Use zookeeper/hearbeating to swap on failure.

So we use zookeeper node(sth that we can achieve consensus with) and all the nodes constantly do heartbeats with zookeeper and the primary matching engine
and then if they all agree that the primary is down, then zookeeper will elect a new leader from the backups.

How do we set up our backup?
- Option 1: for the backups to get the same exact state as the primary node, we require every client to send orders to all backups as well.
This is not good for a couple of reasons:
  - reqs(order reqs) of clients could end up in different orders on nodes. So primary executes client 1 msg then client 2 msg. But backup gets
  client 2 then client 1. Now if primary goes down and we switch to backup, the order of trades changes which is very bad.
- Option 2: State machine replication
  - backups listen to **output** of primary and update their state accordingly.

We mentioned that the primary matching engine is outputting events with sequence numbers, meaning we can apply them in correct order
on the backup nodes. We don't even have to get those msgs in order on the backup, because the outputs might get delivered to backups out of order,
but because the msgs have sequence numbers(we have re-transmitter nodes), the backup is always able to re-arrange them in correct order.

#### Edge case
Let's say the client sends an order to primary matching engine and that one order triggers multiple different trades. So primary gonna
output **multiple** msgs. Now let's say primary matching engine sends out msg #1 and then it goes down before msg #2 #3 delivered.
That's bad because these 3 events are supposed to be atomic. Meaning it doesn't make sense to have msg #1 without #2 and #3.

Fortunately we know all ops are deterministic. Since backup runs the same code as primary matching engine node,
when the backup sees #1 and not #2 and #3, it says: "I didn't even receive msgs #2 and #3 but I know they have to happen at the same time."
So backup is gonna generate #2 and #3 based off of #1.

So for this to work, we need atomicity at the **individual msg level**. So msg #1 itself should be comprised off multiple network packets.
So we can have:
1. a checksum to validate data integrity
2. have an end marker at the end of last msg that shows there are no related msgs for that sequence of msgs.

These two will make sure atomicity at the individual msg level.

### Matching engine partitioning
Up to this point, we had one single matching engine for all stocks.

**We can't partition within the same symbol**, e.g. AAPL. Because then two orders that might have traded, wouldn't see another without 
cross-shard communication. Note that we want all orders for the same symbol to be able to view another.

We could have multiple partitions communicate with each other, but that would incur a lot of latency.

So we shard by symbol. The problem with this is creating orders using two of symbols. For example buy AAPL and sell GOOGL in one order.
Or cancel AAPL order at the same exact time as we cancel GOOGL order.

So we need a strongly consistent distributed transaction -> 2P commit. It's gonna be slow. So we don't offer that in øur exchange in first place.
Because it's gonna make every one else that are working with those symbols to wait for this distributed transaction.

### Returning ordered msgs to clients
For one client, when an op comes in(order/cancel) or is published(trade), it grabs a lock for the clientIds to ensure that it gets
the correct sequence number.

We need rate limiting, auth and ... , so we need a gateway.

Zookeeper does it's leader election from matching engine backups, when the primary goes down.

![](./img/design-stock-exchange/1.png)