https://www.youtube.com/watch?v=ugVwhsWslAc&t=769s&ab_channel=Jordanhasnolife

## Database intro
### Objectives of a database
- fast reads
- fast writes
- persistent(durable) data: How do we do that? Generally we're using hard drives. With hard drives, you always want to aim for
sequential operations. Meaning it speeds things up quite a bit to be accessing data that's much closer on the disk to another.

Note: The disk goes either 5400 or 7200 rotations per minute generally speaking.

Generally, we're using hard drives which results in slow random reads. We should always aim for sequential operations. Much cheaper
than SSDs but slower.

### Naive DB implementation
Literally just a list, O(n) reads and updates. Think of it as an array of tuples. So everytime you wanna read, you have to
search through the array and every single time you wanna update, you also have to search through the array for what you wanna update.
Writes are still in constant time(O(1)).

### Slightly better implementation
Append only log on disk to take advantage of sequential logs. It means you actually overwrite things, by writing an additional entry
in the log. This way you can benefit more from sequential writes.

We add new entries to the end.

### Better database implementation
hashmap, O(1) reads and writes

However, this does not scale because the second there is too much data, we're in trouble, hashmap has to go on disk which becomes slow.

However this doesn't really work well, the second you can't fit your entire hashmap in memory, it's really bad to put a hashmap on disk
because we would have random reads and random writes on disk which are bad because the mechanical arm has to go around and spin all the
time. It means it's gonna take longer to read and write data.

We use a hashing function to map a given key to a certain place in memory.

So while hashmaps are good when there's a small dataset(that can fit entirely on memory), but the second you're dealing with a ton of data
which is what we're worried here, it becomes infeasible.

We spoke about how we can make writing faster -> using append-only log

Let's see how we can make reading faster? We use indexes.

### Indexes - making read times much faster
keep extra data on each write to improve DB read times

- Pro: faster reads
- Con: slower writes(only use indexes if you need them, do not declare an index for every field)

### Types of index implementation
- Hash indexes
- LSM trees + SSTables
- B Trees

## Hash indexes
These are not feasible for DBs unless in rare occasions.

keep an in-memory hash table of the key mapped to the memory location of the corresponding data, occasionally write to disk for persistence

The entire point of hash index is that for the field you're indexing on, you take the key and then you map it to the offset on disk. So that
way we can literally do an O(1) accessing RAM.

Note: Hashmaps do very poorly on disk because we're gonna do random access and random write on disk which would be very bad. So store hashmaps
on RAM.

Pros: easy to implement and very fast(disks are slow, RAM is fast)

Cons:
- All of the keys must fit in memory
- bad for range queries. For example you wanna quickly find all the keys with a given range of values, they're gonna be scattered all around
the disk and that's gonna be very inefficient. Because then you have to keep doing more random accesses on disk.

## SSTables + LSM trees
More feasible for DBs

Write first goes to an in-memory balanced binary search tree(memtable). In other words, first we write to an in-memory buffer. On RAM,
we would have a self-balancing tree like a red black tree or an AVL tree and it's called the memtable - our LSM tree. When the tree becomes too large,
you take all the contents of it which should be automatically sorted by virtue of using a tree traversal and you would write them to
sth called a SSTable file and that SSTable file would be held on disk. In the event that the DB crashes, obviously whatever is in memory
is gonna be lost, so we keep a second log called a write-a-head log expressing all of the changes that we have in the tree and that
way if the machine crashes, you can easily restore that tree.

Note: once you write the tree to disk, you reset the tree to nothing again and you start writing keys in there.

When tree becomes too large, write the contents of it(sorted by key name)

### SSTables and LSM trees continued
On SSTables, the keys are sorted and next to keys, are the values(look at the img).

Since we're using append-only logs, there will be duplicate keys.

We might have multiple of these SSTable files. Because every single time that LSM tree gets too big, we write it to an SSTable.
Another thing to note is you might have duplicate keys between SSTable files. Since everything is an append only operation,
if you want to overwrite a key, it might just go into a newer SSTable file. For example in img, the same key 33 in SSTable 1 has value of
Jabbar but in SSTable 2, it has value of Pippen.

But the issue is: By virtue of having all of these duplicate keys, we're potentially wasting a ton of storage. If you're updating the
values in your DB a lot, you might be using a ton of space that you could actually get rid of, if you were somehow compress these SSTables.

So that's exactly what we're gonna do. We're gonna turn it into a compacted SSTable and it means all of the duplicate keys are going to have their
updated value(because that's the correct value) and additionally all of the other keys are also present. How do we merge these together? It's really efficient.
Remember merge sort, you start at the top of the each SSTable. For example in img, we start with Westbrook and Iverson and say: 0 for Westbrook is smaller
than 3 for Iverson, so Westbrook is gonna be at first in the compacted SSTable. Then 3 of Iverson is smaller than 7 for Anthony, so that's
gonna be next in the compacted SSTable and ... . This is a linear time operation(O(n)) to compress these tables and so that's a process
that often gets run in the background to optimize the storage space.

So newer SSTables gets more precedence over old ones, because we want the updated values for duplicate keys between tables.

Let's discuss how to quickly read a value by it's index!
![](./img/database-design/1.png)

We discussed how to write to SSTables, now let's discuss how to read?

When you're looking for a key, first you would query the memtable. We would literally just do a binary tree traversal and look for the key in there
which is O(log n). Secondaly, we would look at each SSTable in order from newest to oldest looking for the key. The issue with this is that
fundamentally, you might go through SSTables until you have none left and if there are a lot of them and the key is not in any of them,
you've wasted a ton of time. There's an optimization called bloom filters that optimizes this, so in practice it's not actually so brutal if the
key doesn't exist in SSTables, but even still, from a theoretical perspective, it's bad on reads.

Another optimization to that is the fact that all of the SSTables are sorted. For each SSTable keep an in-memory sparse hashtable so that we can
quickly search them.

For each SSTable, have a sparse in-memory hashmap of keys with their value in memory. Since each table is sorted, we can quickly binary search
the SSTable to find the value of a key.
As you can see, in hashtable, we have the memory address. Now let's say we wanna find Andy in SSTable. We know that Andy's key is between Alice and
Bob(look at hashtable) because the SSTable is sorted. So we would start at memory address of Alice and start at memory address of Bob and run a
binary search(splitting the half in each step - O(log n))
![](./img/database-design/2.png)

### SSTables and LSM trees summarized
Pros:
- high write throughput due to writes going to in-memory buffer - writes are fast
- good for range queries due to internal sorting of data in the index. So if we want to get all of the values for a range of keys,
it's fast, because ranges of keys are stored together in SSTables since they're sorted

Cons:
- slow reads, especially if the key we are looking for is old or does not exist
- merging process of log segments can take up background resources

Note: Writes similar to reads in memory, are faster than disk because memory is much closer to the CPU

## B-Trees
The entire point of a B-Tree is to model your data such that it is a tree on disk.

Let's say we have a bunch of names and we wanna be able to find the corresponding age. As you can see, we wanna keep all of my data
in order(for example on first level: A - E - P - Z) and then have pointers(`REF` in the img) down the tree, to show where the actual data is.

At top level, we have a reference between all keys that go from A-E, a reference from all keys that go from E to P and all keys that go from
P to Z. Let's say we want to find the name Thomas. First we take the REF in P-Z, moving on to the next page, then we take the REF between T and Z,
finally I would ultimately go to the page where values are actually stored. All this is done in `O(log n)`.
![](./img/database-design/3.png)

### B-Trees continued
To read: traverse through the tree and find the value

To Update: traverse through the tree, find the value and then change it

To write: traverse through the tree, if there is extra space in the block where the value belongs, add the key, otherwise(if one of those
layers in B-Tree is out of space and there's no room to add the key and it's corresponding value) you have to
split the location(page) block in two, add the key, and then update the parent block(page) to reflect this action(reference both of those
blocks). This process can be made durable in the event of crashes using a write ahead log. In other words, if during this process(splitting the
pages and updating the parent) the server crashes, you have to be able to restore that, so we can use a write-ahead-log(similar with LSM trees) where
you write all of your changes down tentatively before you make them, so that in the event of a crash, you can restore that.

### B-Trees summarized
Pros:
- relatively fast reads, most B-Trees can be stored in only 3 or 4 levels(reads are O(log n))
- good for range queries as data is kept internally sorted. All of the keys that are next to one another in the actual range, are physically
next to one another on the drive

Cons:
- relatively slow writes, have to write to disk as opposed to memory

## Conclusion
In a system, it is important to know what type of database engine/design you are using so that you can optimize for writes or reads.

- Hash indexes: fast but only useful on small datasets. So maybe sth like a redis DB where everything is already in memory
- SSTables and LSM trees: better for writing, slower for reading. When you have to write a lot and quickly but not great for reading because
you risk having to go through a bunch of SSTables
- B-Trees: better for reading, slower for writing(because you're not writing to an in-memory buffer)