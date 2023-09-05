https://www.youtube.com/watch?v=gB7qazeSD3k&ab_channel=Jordanhasnolife

## Two phase locking
Two phase locking is just another method of achieving isolation/serializability which means make concurrent transactions seen
as if they were running on one thread(one at a time.)

So: Two phase locking means make concurrent transactions seen as if they were running one at a time.

We've seen how we do this by literally running the transactions one at a time in terms of actual serial execution, however for today
we're gonna figure out a **better** way of doing that by using a bunch of locking.

Typically two phase locking is sth that has been used used in the past because actual serial execution wasn't feasible(CPUs weren't fast
enough) and so you'll see this in a lot of traditional SQL DBs.

In the past(look at race condition vid), locks basically means exclusive access to this piece of data. I couldn't read a row unless I 
grabbed the lock, I couldn't write a row unless I also grabbed the lock. But in two phase locking, instead of a lock enabling you
exclusive access to the row itself, you may have a concept to share the access. In two phase locking, we have a reader lock and also
a writer lock. So consider it the same lock but it's got two modes.

If we want to read this row, we can grab the lock in reader mode and if then I want to write to it, we have to upgrade that lock to
writer mode.

**Note: Multiple threads or transactions can be reading a row, but in order to write to that row, you have to be the only one grabbing the lock.
You need to wait for all of those readers to leave and then upgrade your access to exclusive.**

The reason this works is that for sth like a read-modify-update cycle which is the cause of many different race conditions within DBs,
we know that our predicate aka the read, is always gonna remain valid because of the fact that whatever rows we're reading in our
predicate, can't be modified because no one is going to be able to upgrade to get the writer lock because we're concurrently reading them.
So again it means the predicate is valid hence this two-phase locking approach actually works.

So for read modify update transactions, we know the predicate will be valid.

But we have two problems with this approach: Deadlocks and phantoms

## Deadlocks
You might say: "oh, if everyone grabs the resources in the same order, it's impossible to have deadlocks." No. That's not necessarily the case.

Example of deadlock: Transaction 1 wants to read row 1 and row 2 and then write some data of row 2 to row 1. So it grabs reader lock on row 1 and 2.
Simultaneously, transaction 2 wants to read row 1 and 2 and add some of the data of row 1 to row 2. So it grabs the reader lock on
row 1 and row 2.

Now **neither** of transactions can grab the writer lock on the row they want. Because we can't grab a writer lock unless you are
the only person(transaction) that's trying to write to that row. So here we have a deadlock. Transaction 1 can't complete it's write
until transaction 2 completes his write and transaction 2 can't complete it's write until transaction 1 completes it's write.

So we need to somehow avoid deadlocks.

Two phase locking is slow, because:
There are too many deadlocks. Deadlocks make two-phase locking slow. Why? Because the system has to detect them and abort one of
the transactions that's part of the deadlock and then run it again.

In the example we discussed, system has to pause the transaction 2 and run transaction 1 to completion and then once that's done,
we can run the transaction 2. It's gonna take quite a bit longer, because we're no longer running them concurrently.

## Phantoms
In addition to deadlocks which are a big problem with two-phase locking because it takes a lot of resources to grab the lock and everything
is defensive as well because we're grabbing locks where we don't necessarily need them and also detecting deadlocks and running transactions
involved in deadlocks in sequence.

Still we haven't cover all of our race conditions, still our system is not perfect. Let's remind ourselves about phantoms.

Phantoms are when you do a read-modify-update cycle and then update part is actually adding new rows to the table.

For example let's say rows that have: `where class_name = 'flexibility and class_time = 6` currently are locked in reader mode.
But we do a `SELECT` query(we can do a select although having the rows locked, because it's just a read), but inside our select, we do an insert.
But there's no locks prohibiting us from writing because there's no locks to be had on the rows that are gonna be inserted - they didn't exist yet.

So we need to we need to be able to lock rows that don't even exist yet. To fix this, we use predicate locks. 

### Predicate locks
Predicate locks allow us(in certain DBs) to grab all rows that fit certain conditions. So before we make our write to the table, we put
a lock on all the rows that fit a certain condition. For example rows that where their `class_name = 'flexibility'`. Now if the row
doesn't even exist yet, it would still have a lock on it, because it fits those conditions.

The issue with predicate locks is that they're slow to run. Because we have to evaluate the query to find the rows that fit those conditions
and if there's not an index that corresponds with this query, this could be very slow to run.

### Index range locking
Take advantage of table index to grab **more(superset)** predicate locks than necessary. So sometimes it's faster to lock a superset of the rows,
so we're locking more than necessary because running the query to figure out which rows to lock, is a little bit faster.

Let's say we want to lock the rows that fit this condition: `where class_name = 'flexibility and class_time = 6` but we only have an index
on class_name. So index of querying with the mentioned condition which has a column that doesn't have an index, we search the rows
using the columns in the condition that does have an index, in this case, the condition we would search is: `where class_name = 'flexibility`.
So now we would potentially grab more locks than we need. So we find the rows with this new condition in `O(log n)` time simply by virtue of the
fact that there is an index on that field. So now we lock more rows than we actually need to because it's a little bit faster
to run that query. This does come with one disadvantage of the fact that now if we're locking more rows than we need to,
it's possible that other queries(transactions) that didn't have to be blocked are now going to be blocked.
So you wanna be careful about doing this, for example you wouldn't wanna block the whole table just to run a very small query, because
then we're gonna be blocking all transactions of the table.