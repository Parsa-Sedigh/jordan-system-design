Rule: The trade happens when the highest bid >= lowest ask(meaning the spread has crossed). The spread is difference between
lowest ask and highest bid. So when they cross, it means someone is willing to pay >= the amount that someone is willing to sell at.
The trade is initiated and the volume is removed from the order book.

Rule: If one side was already in the order book, the price of the trade is the price of order **already** in the book.

### Order book interface(for now)
There are multiple impl of an order book.

- placeOrder(order): takes an order obj and either fills it or places it in the limit book, prints trades that have taken place as a result
- cancelOrder(orderId): takes an orderId and cancels it if it has not yet been filled, otherwise is a no-op
- getVolumeAtPrice(price, buyingOrSelling): get volume of open orders for either buying or selling side of the order book

### Data structures for limit order
Flow of submitting a buy order:
- first, check the lowest price of the sell side of the limit book
- if the buy price is >= lowest sell, execute a trade
- if buyer still has more volume left to fill, look at the next lowest sell price and keep going
- if there's unfilled volume for the buyer's trade, add it to the buyer heap

What DS for storing the order book? A heap! O(log(n)) inserts and pops.
- sell side: min heap
- buy side: max heap. So we can quickly see what is the highest price that someone is willing to buy

### Inspecting the heaps
However, it's not that simple! We can't just have some order nodes in the heaps.

We know that we can have many orders with the same price. We said that we would execute those in order of timestamp.
**So we actually make our heap to hold a bunch of queues for orders.**

For this, we have a couple of options:
1. make the heaps based on **both price and timestamp**.
2. It's algorithmically faster, if we were to use queues as heap nodes. So we would have a heap of queues where the queues are
representing one price level and inside the queues, we have the order nodes in the timestamp order.

So now we satisfied all the requirements. We can look through the price levels fast and also if there are multiple orders with the same price,
we sort them by the creation time.

### Getting volume
We need to be able to get volume, but it's important that this call happens as fast as possible. Right now, we'd have to loop through
every el of a queue to sum up it's volume. But what if instead, we kept a hashmap that kept track of the volume at each price(price level),
and incremented/decremented the volume counter when orders were added and cancelled from the limit book?

Then we would have O(1) time complexity for returning volume.

### Cancellations
Two possible ways to cancel an order:

#### **actively** remove order from the limit book:
Extra time complexity incurred by actively removing nodes from our queue and heap. So now the n is smaller in (O(log(n))) for future ops.
But the negative point is removing a node from queue is O(n).

Note: Heap node(queue) is removed from heap if queue is empty, representing that price level is no longer in the heap.

Note: queue time complexity can be negated if instead of using a queue with linked list impl, we use a queue implemented via doubly linked list and hashmap.
Therefore, we can use the hashmap in O(1) to get the location of the order in node deque and then remove it in O(1) again.
But the removal of a queue from heap(because there are no orders in that price level), is still O(log(n)).
Note that here we're assuming that we know the location of the queue in the heap, which is only possible if we store a hashmap and have a pointer to
that queue).

Why a doubly linked list instead of linked list?

Because we want direct pointer to the node we want to delete (via the global order_id → Order* hashmap), 
and each Order carries both prev and next pointers. Deletion is therefore three pointer swaps(when having doubly linked list).

#### lazily **mark** order as cancelled, and if we come upon it while trying to execute a trade, we skip over the cancelled order
With this, we don't need to deal with removing nodes time complexity. But since the 

If there are many cancellations, we may have to skip a lot of potential orders(because they were cancelled) when trying to fill a new order
if we're not removing nodes(the nodes in order book heap are queues) from the heap actively, there will be more nodes in the heap and
adding and removing nodes from the heap will take longer as a result


> Order book:
> 
> private minHeap bestAsk
> 
> private maxHeap bestBid
> 
> // allows us to find orders fast
> private hashmap<int (orderId), Order> orderMap
> 
> private hashmap<[double (price), buyOrSellEnum (side)], int (volume)> volumeMap
> 
> // allows us to find queues that are in heaps fast
> private hashmap<[double (price), buyOrSellEnum (side)], PriceQueue> queueMap

PlaceOrder(Order):
> oppositeBook = bestBid if order.side == Sell else bestAsk
> 
> sameBook = bestBid if order.side == Buy else bestAsk
> 
> orderMap[order.orderId] = order
> 
> // go through order book and fill the cur order as long as there's matching order in the opposite side
> while order.volume > 0 and oppositeBook not empty:
>   if not (order.side == BUY and oppositeBook.peek().peek().price <= order.price) or
>      not (order.side == SELL and oppositeBook.peek().peek().price >= order.price):
>        break
> 
>      otherOrder = oppositeBook.peek().peek()
>
>      tradePrice = otherOrder.price
>
>      tradeVolume = min(order.volume, otherOrder.volume)
> 
>      otherOrder.volume -= tradeVolume
> 
>      print(Made by {otherOrder.client}, taken by {order.client}, {tradeVolume} shares @ {tradePrice})
> 
>      if otherOrder.volume == 0:
>           cancel(otherOrder) // same as removing
> 
> // # if still volume left → add to book
> if order.volume > 0:
>   addOrderToBook(order, sameBook)

```python
def addOrderToBook(order):
    if not (order.price, order.side) in queueMap:
        # queueMap is the actual price level
        queueMap[(order.price, order.side)] = new DoublyLinkedList(order)
        volumeMap[(order.price, order.side)] = order.volume
    else:
        queueMap[(order.price, order.side)].push(order)
        volumeMap[(order.price, order.side)] += order.volume

```

```python
def cancelOrder(orderId):
    if orderId in orderMap:
        order = order_map[orderId]
        if not order or order.remaining <= 0:
            return False  # already filled or cancelled
        
        price_level = queueMap[(order.price, order.side)]
        
        # O(1) - assuming price_level is a doubly linked list
        price_level.remove(order)

        # If this price level is now empty → remove from price heap
        if price_level.len == 0:
            # find out which book side we're concerned with?
            sameBook = bestBid if order.side == BUY else bestAsk
            
            # If associated queue got empty, remove it from the order book heap
            sameBook.remove(price_level) # O(log(n))

        # Don't use order.volume, use remaining
        volumeMap[(order.price, order.side)] -= order.remaining
        order.remaining = 0
        orderMap.delete(orderId)
```

```python
def getVolumeAtPrice(price, side):
    return volume_map[(price, side)] or 0 if not exists
```

### market orders
- just say buy or sell, don't specify a price
- go through the opposite side of the order book heaps until volume for trade is exhausted, and then do **not** store trade in limit order book

### Trigger orders
