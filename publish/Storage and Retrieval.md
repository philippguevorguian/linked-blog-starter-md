# The index tradeoff
If we don't index database entries, then we have excellent write operation performance, because we only have to write a key-value pair. However, on reading a specific file, we have poor performance since we need to search for the key in the entire database which takes O(n) time, where n is the number of keys we have in the database. Adding an index abstraction layer reduces performance of writes because we always have to update the index values when writing new data, but it improves read operation performance since we can use the index to perform *lookup* operations. 

## Hash Index
We can keep an in-memory *hash map* that maps each key to a *byte offset* within the database. The values do not have to be stored in memory, only the key and byte offsets. So the entire index is available in memory the whole time, this means that the operations are going to be fast, but we can't keep a lot of data in memory. Thus, this approach is most effective when we want a high throughput for operations but don't have a large collection of data. This is especially advantageous if we have few individual records of information (smaller hashmap), but the individual values are large(keep that stuff on disk).

### Database and Disk Space
If we continually append to the database, we may eventually run out of disk storage. We can use compaction to fix this. Each segment of the database will have it's own seperate in memory hashmap loaded and used to make operations. If we have m segments, then we are able to make reads in O(m) since we may have to access each available hash map until we find the right one and make an O(1) read (remember, this lookup is an in-memory operation). 

### IRL considerations
- File format
- Deletions
- Crash Recovery
- Partially written entries
- Concurrency control
### Why append instead of overwriting in place?
- Due to HDD and SSD architecture, sequential writes are faster due to contiguous storage of memory. 
- Crash recovery simpler since we have redundancy of written records.
- Merging old data means it doesn't get fragmented over time

### Limitations
- Hash table does have to fit in memory
- Scanning over many keys doesn't work
	- that is to say, if we want to query a range of keys, we need to individually query each key in the hashmaps

## SStables and LSM Trees
## What is an SStable
 A *sorted string table*, in other words: a table in which the keys are sorted according to their string characteristics (probably alphebetical).

### Merging SStables
Way easier. We use a *mergesort* on table records and call it a day. Read the input segments first key and copy the last key. We then merge on the keys. The output file with merged keys is therefore already sorted. A long hard look at the below figure is mandatory:

![[Pasted image 20220921211553.png]]

### Non-comprehensive in memory mapping
That is to say, not every key value needs to be mapped in-memory to a particular index. This is because we can leverage the alphebetical key sorting intrinsic to the data structure. E.g. we have the keys "Belize" and "Belugar" for users search results in some application. We know that the key in the SStable for "Bell" will be between those two keys. We can now scan keys from belize to beluga to find it where the time it takes to get to the correct key will depend on the sparsity of our index mapping. It is recommended that we index one key every few kilobytes.

### Save space and IO bandwidth
We need to scan through keys on read operations remember, since we don't index every single key-value pair. Ok, this kind of grouping means we can actually group records together and compress them when we want to write to disk. Now, each entry of the sparse in-memory index points to the byte address that corresponds to the start of one such compressed block. This saves space and *IO bandwidth*. The book doesn't mention this but the obvious tradeoff is that our system needs to execute compression and decompression algorithms which cost compute power. Granted, the books mentions in a prior chapter that most tech stacks aren't compute limited.

### Constructing and Maintaining SSTables
We've taken for granted the fact that using this approach, we have our keys sorted in e.g. alphabetical order. However, this is a non-trivial consideration since this is more difficult to accomplish on disk (*B-trees*). However, this becomes much more feasible to accomplish in memory. In the case of in-memory we can use *red-black trees* or *AVL trees* to ensure our index remains sorted: 

To go into additional depth:

#### Red Black Trees
Red black trees are *binary search trees* that use an extra bit of information to store a "color" state on each node which is actually just a binary variable that enforces additional requirements for this data structure. 
The full list of requirements of a red black tree are as follows:
1) Every node is either red or black (binary variable as mentioned)
2)  All NIL nodes are black. NIL nodes are nodes which are not storing information.
3) A red node does not have a red child.
4) Every path from a given node to any of its descendant NIL nodes goes through the same number of black nodes. 

![[Pasted image 20220909110602.png]]
We can see that this is a valid red-black tree since from node 13 to any of the NIL nodes, there is one transitory black node.

For reading/tree traversal, this additonal information doesn't change anything other than a minor memory overhead (1 additional bit per node). For writing/deleting operations, requirements 3 and 4 require additional computation to maintain, with six different potential cases for each (both reading and writing) that are addressed by specific algorithms that can balance the tree, if you want to know these in detail, go to wikipedia. If you don't know what balancing a tree is, you're a *scrub*. 

#### AVL Trees
An AVL tree is another type of self-balancing binary search tree. In this case, the requirement that maintains the structure of the tree is that the heights of child subtrees may differ by at most one. This is accomplished by holding a 2-bit variable called the *balance factor* of each node which is defined as the difference in height between the right and left subtrees of that node. 

As with red-black trees, reading operations are unaffected by this property but height/deletion operations are. Namely, if subtree A has height 6 and subtree B has height 7 and e.g. a write operation changes subtree height B by +1, then the balance factor is no longer within the accepted range and rebalancing of the tree will be necessary. In the case of the AVL tree this is accomplished by simple or double rotations. See an example of a simple rotation which balances an AVL tree below:

![[Pasted image 20220909112823.png]]


#### Tree Sort
*Tree Sort* is what allows us to leverage the tree structures to maintain a sorted index.  Actually, tree sort can be implemented for self-balancing trees and non self-balancing trees but with worse time complexity. With a self-balancing tree (such as those described above) we get O(n log(n)) worst case. Below is the pseudocode for tree-sort that allows our in-memory index to be sorted at all times. This is relatively low level and not described in the book but I think it's cool and integral to the database design:
```
 STRUCTURE BinaryTree
     BinaryTree:LeftSubTree
     Object:Node
     BinaryTree:RightSubTree
 
 PROCEDURE Insert(BinaryTree:searchTree, Object:item)
     IF searchTree.Node IS NULL THEN
         SET searchTree.Node TO item
     ELSE
         IF item IS LESS THAN searchTree.Node THEN
             Insert(searchTree.LeftSubTree, item)
         ELSE
             Insert(searchTree.RightSubTree, item)
 
 PROCEDURE InOrder(BinaryTree:searchTree)
     IF searchTree.Node IS NULL THEN
         EXIT PROCEDURE
     ELSE
         InOrder(searchTree.LeftSubTree)
         EMIT searchTree.Node
         InOrder(searchTree.RightSubTree)
 
 PROCEDURE TreeSort(Collection:items)
     BinaryTree:searchTree
    
     FOR EACH individualItem IN items
         Insert(searchTree, individualItem)
    
     InOrder(searchTree)
```

We will just be using the Insert function and the self-balancing tree will take care of the rest. 

#### Implementation
In applying either of these data structures, we are able to write in keys to our storage engine in any order and still read them in sequential order. To implement this:
- Whenever we process a write operation, we add it to an in memory self balancing tree such as those described above. This in memory tree is idiomatically referred to as a *memtable*.
- When the memtable exceeds some memory threshold (a few megabytes), we write it out to a disk as an SSTable file. We can perform this efficiently since the self balancing tree has already sorted the entries by key. This SSTable file is thus the most recent segment of the database. We have the ability to simultaneously write out that SSTable to the disk and write to a new memtable (self balancing tree) instance for the subsequent upcoming segment.
- To serve a *read request*, we first search the memtable, then the most recent segment then the second most recent, etc.... Since we want the most recent database value for a given key.
- Ocassionally run a merge and compaction process in parallel to combine the SSTable on disk files and discard overwritted/deleted values.

The only disadvantage to this architecture is that if the database crashes at any given time, the information stored in the memtable that has yet to be written to disk is lost. To deal with this, we an maintain a seperate log on the disk where we immediately append every single write without any fancy rablancing. It's ok that the structure of this log isn't sorted since we only have to use it to restore memtable values during crashed. Once a memtable has been completely written to disk, we can clear/delete the corresponding log since the data is now on disk in a sorted SSTable file as a segment in our storage engine.



### Making an LSM-tree out of SSTables
A *log-structured merge-tree* can be generated from an SSTable. This is implemented by LevelDB and RocksDB and similar applications in Cassandra and HBase. Lucene stores a full text index rather than a key-value index but the same principles are applied.

#### Performance Optimizations
We have a barrier to performant implementation of a lookup operation if it doesn't exist in the database as it has been dsecribed so far. If the key isn't in the database, we must check the memtable, then the most recent segment, then the second most recent segment then the third most recent segment and so on until we get to the oldest segment that hasn't been compacted/merged away. To deal with this we can use *Bloom filters*. Bloom filters are a memory efficient data structure that can tell you if a key does not appear in the database, and now you're blessed.

##### Bloom filters
This data structure is too proper. You start with an array of m bits set to 0. We also need some k different hash functions. Every time that we add some item to our database, we hash it using all k of the hash functions and recieve an array of k index positions within m. At all of these index positions, we set the value of the bits to 1 instead of 0.

Now we want to know if some element we are looking for has been seen by this Bloom filter (and thus is in our database or whatever), we just hash it and check all returned k indices to see if they are **all** equal to one. 

![[Pasted image 20220909120956.png]]

But wait, isn't it possible that some other combination of items could make the indices all equal to one? Yes but it is unlikely to happen if we pick sufficiently large amount of k hash functions that are independent. "For a good hash function with a wide output, there should be little if any correlation between different bit-fields of such a hash, so this type of hash can be used to generate multiple "different" hash functions by slicing its output into multiple bit fields. Alternatively, one can pass k different initial values (such as 0, 1, ..., k − 1) to a hash function that takes an initial value; or add (or append) these values to the key."


#### Merging / Compaction

Merging and compaction algorithms are also design choices. Includes *size-tiered* and *leveled compaction*. Guess which one LevelDB uses lmao. Size tiered compaction means newer SStables are smaller and get compacted into older and larger ones. "Level tiered compaction means older data is moved into seperate levels **what is this??** which means the compaction can proceed more incrementally and use less disk space"

#### Overall evaluation of LSM tree/SSTable method
It's proper. Works well with big data, data is sorted allowing for ranged queries, and you get high throughput. 

## B-trees
![[Pasted image 20220921215117.png]]
Most widely used type of index. Also use key sorted key-value pairs. However, b-trees do not use variable size segments as the SSTable approach does. Instead, we have fixed-size blocks or *pages* 4kb in size (sometimes bigger), and read or write one page at a time. This design matches underlying hardware design of disks which are arranged in fixed size blocks of storage. Each block has a corresponding address which allows linking of pages to each other. These page references can build up a tree of pages.

One page is the *root of the B-tree*. If we want to find the value of any given key we start with this root page. This root page has several keys and references to child pages which themselves also hold references to their own child pages. The keys between the reference tell you the lower and upper boundary for keys and references available with the page that corresponds to that reference.

In traversing down this hierarchal structure, we will eventually get to a *leaf page* that only stores key-value pairs. At this level, we can access the value for our key directly. If we want to write to a given key we do the same thing and write over the previous value in place. Cool, but now let's consider a situation where we want to write a new key. This may be perfectly fine if it fits within a page as the key-value pair may just be inserted into the appropriate location. 

Complexity arises when we have to write a key-value pair to a page that doesn't have additional space. In this case, we must split the leaf page in half, write the new key-value pair into one of the halves and update the parent page of this leaf page such that it references the updated boundaries generated by splitting one leaf page into two:

![[Pasted image 20220921215200.png]]

By doing this, we ensure that the b-tree remains balanced and always has a depth of O(n). Typically, we only need 3/4 levels of depth in most use cases. A four-level tree of 4kb pages with a branching factor of 500 can store 256 terabytes. 4000 bytes x 500^4 leaf pages (four levels not including the root)= 2.5 x 10^14 bytes or ~250 terabytes for some back of the napkin evidence. 

### Making B-trees reliable
Note that in some cases, we will have to overwrite several different pages in place. For example, consider the case of splitting a page in two to accomodate for space on an insert operation. We have to write over both pages that are the result of the split, as well as update their parent.

(and this isn't mentioned in the book but as I understand if our update to the parent page then leads to excess space being occupied there, then this will necessitate splitting the parent page in two and updating that parent's page as well and so on until a page does have space). 

This is of course kinda sus since if the database crashes in the midst of such an operation then we might leave the database in a state where a page isn't indexed by its parent and is just floating without any way to access it from the root page. In order to prevent this we can just maintain a *write ahead log* or *redo log* which is an append-only file that stores every b-tree operation. This file can be used to restore the b-tree to a consistent state in the case of a crash. 

Concurrency also has to be managed tightly since two operations managing related pages at the same time can lead the database to an inconsistent state. We can prevent this with *latches*. Latches prevent concurrent access to RAM structures by kernel code. 

### B-tree optimization
- Nize the write ahead log and just copy-on-write. Modified pages get written to a diff place and a new version of the parent page is created referencing the old one. After this is established to be consistent just delete the old parent and child pages. 
- Abbreviate keys, they only need to provide boundaries between key ranges and nothing else. Doing this means we can fit more keys on a single page and thus reduce our branching factor on the b-tree.
- Attempting to get pages to be stored sequentially on disk **somehow**
- Sibling page reference to scan across key-ranges without referring to their parent page. This means that at the beginning and end of a page, we have pointers that reference the page before / after it.
- Fractal tree implementation
	- Also called a fractral tree index and has nothing to do with fractals.
	- B-tree performance but asymptotically faster insertions and deletions
	- "unlike a B-tree, a fractal tree index has buffers at each node, which allow insertions, deletions and other changes to be stored in intermediate locations. The goal of the buffers is to schedule disk writes so that each write performs a large amount of useful work, thereby avoiding the worst-case performance of B-trees, in which each disk write may change a small amount of data on disk"


## B-trees vs LSM-Trees

Basically, LSM trees can be better for writing operations due to configuration dependent low write amplification and write SSTables sequentially which is faster than random writes for larger sized b-tree page files. This is less advantageous on SSD hardware, since the firmware uses log-structured algorithm to convert random writes into sequential ones, but *write amplification* and *fragmentation* still matter.

On the other hand, LSM tree approach can mess with throughput due to the whole merging/compaction process running in parallel. The bigger the database the more compaction. If you messed up really big time, your compaction and merge rate can't keep up with disk writes. In this case you need to manually interfere. 

"An advantage of B-trees is that each key exists in exactly one place in the index, whereas a log-structured storage engine may have multiple copies of the same key in different segments. This aspect makes B-trees attractive in databases that want to offer strong transactional semantics: in many relational databases, transaction isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks can be directly attached to the tree"

## Other indexing structures
Examples above have been primary index oriented. This primary key uniquely identifies the entry within the databse. Any record within the database is able to use this key to get the corresponding value. However, we may also have secondary indexes, which are often used for joins efficiently in relational data models. These secondary indices are however NOT UNIQUE intrinsically. They can be made unique by just appending a row identifier to it or they can making each value in the index a list of matching row identifiers. 

### Storing values in the index
The key in the index is what we use to identify a given piece of data, but the value within the index can actually be two different things. The obvious case (to me) is that the value corresponding to a given key is the actual information that we are looking for when we make a query. The other option is to actually have the value hold a reference/pointer to another file where the data is stored. This other file is typically called a heap file which is not usually particularly organized and it is basically super messy since we have a reference to the location of our needed information. This means we don't have to duplicate data when we want a lot of secondary indexes: every secondary index just has to reference a location in the heap file and the data just chills there. 
Sometimes, this extra redirect after querying (from the pointer stored in the value to the heapfile) creates a performance overhead that is not appropriate for the given use case. In that situation we use a *clustered index*, where the information we are interested in is stored right in the value and all the secondary indexes refer to the primary index and NOT a heap file.

We can actually find a middle ground here by using a covering index, which stores **some** of the table's columns in the index. Now, some queries can be accessed with the index alone (the index covers that particular query). 

### Multi-column indexes
So far we have a single key to a value. This is a limitation if we want to query two different variables at the same time. With a single key to a value, **we are able** to filter on one field and then subsequently query on another. But in terms of performance, this is simply two seperate queries (even though the second is likely querying less data than the first).

But we can use multi column indexes to address this issue in cases where we anticipate that we are going to want to query across two variables simultaneously. 

#### Concatenated Index
One example is a phone book, where we want to query by first and last name simultaneously to quickly get the phone number of a given person. In this simple string index scenario, we can simply concatenate first and last names to make a new field "fullname" which woud allow a user to query directly for "will young" rather than getting a list of many wills ("will honcharuk", "will young","will lennox") and querying again 

A good example is for a map service that needs to query a given square area within the frame for locations (lke restaurants or whatever).

The query we would want to make is:
``` SQL
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
AND longitude > -0.1162 AND longitude < -0.1004;
```

So in terms of implementation options here:
#### Space filling curves
In trying to convert a two dimensional measure into a single number we can map these two dimensions onto a single axis using space filling curves:
![[Pasted image 20220912185949.png]]
We simply start by bisecting the quadrants of the space along each axis, then iteratively generate quadrants and bisect them with our curve until we have the necessary resolution for our use case. This appears mathematically trivial to me, at least until we reach a large number of points. In the end, every 2d point has a unique and deterministic mapping on a 1 dimensional scale. Should we want to implement this programmatically we can use https://pypi.org/project/hilbertcurve/. 

Note that this approach scales to n dimensions.


#### R-Trees
I personally went and looked into this since it wasn't described in the book, enter at your own discretion, this data structure is kind of sus:
![[Pasted image 20220912183833.png]]

Basically, each node within the tree stores two pieces of info:
1) reference to child nodes
2) Bounding boxes of the child nodes
When we query the R-tree, we check if the query box intersects with R1,R2 or both and go down to their child nodes and see whether this is the case for R3,R4,R5 if it intersects R1 and R6,R7 if the query box intersects R2.

The point is that if we have some query box that exists within the union R1 U R2, we (probably) don't have to touch most of the nodes. The hard part is designing an r-tree which is balanced, doesn't cover empty space and has few overlapping bounding boxes. I won't be going into insert/delete/search algorithms because I don't want to.

#### Full-text search and fuzzy indexes
If you want to return similar keys such as mispelled words or variations on a word. One approach is to use a sparse collection of keys in memory, another is to have the in-memory index be a finite state automation over the characters in the keys, similar to a trie. The automaton can be transformed into a levenshtein automaton, which supports efficient search for words within a given edit distance. Finally, we can of course use classification / machine learning. 

#### Everything in memory

RAM is getting cheaper and we have battery powered RAM which can persist in the case of an outage. You can still write to disk to have a log that you can use to restore the in-memory database but that's it. The main advantage is that you don't have to encode data in structures that are able to be written to disks, i.e. we can just keep our data in fucked python objects native to the library / framework we're using. 


# Transaction Processing or Analytics
## intro
Basically, in many businesses we have to use data for two things:
1) Some form of transaction processing, online transaction processing (OLTP)
2) Analytics

Transaction processing generally refers to computation which the business considers critical and is necessary to it's immediate function. For instance, a bank might need to use customer info to process a transaction (xD). On the other hand, the bank may also want to do some research regarding these transactions to establish trends, evaluate the performance of transactions etc...

Differences between these characteristics are summarized in the table below:


![[Pasted image 20220920163114.png]]

## Data warehousing

Seperate from the bustle of the OLTP systems which are always running, the data warehouse is just chilling, storing the data and not interfering with the operations of the businesses OLTP systems. Data is read-only, goes through an extract-transform-load pipeline and then gets interpreted. 

### What's the diff?
Actually, similar to OLTP, there is probably an SQL-esque interface for the warehouse. But, the architectures of software built to support one or the other actually difffer due to the differences in the image above. The one mentioned that people actually have heard of is spark SQL. 

## Stars and snowflakes: schemas for analytics

Data can be modelled bare different ways. In analytics, there are two models which rain supreme. In the star method, we have a fact table which is the most high level overview of the data and the values in the columns of the fact table are references to entries in dim tables. This is a star because there is a center table (the fact table) and it is connected to a bunch of points (dim tables). The snowflake is like the star but each dim table also has references to further dim table creating a hierarchy of tables. Clearly, snowflake model is more normalized but star model is often preferred since it's easier to work with. 

## Column-oriented storage
If you have a shit ton of data (on the order of petabytes) then it is much more difficulty to store and query them. This is more likely for fact tables than dim tables since fact tables tend to be bigger. They may have many many columns but typically you only want to query a few of them at a time in an analysis context. In OLTP databases, this data is arranged in a row-oriented format. In this case, it's necessary to load every row and then filter out the rows that don't meet the relevant conditions. 

Column-oriented storage means that we store all the values in a column together, store each column in a seperate file and query them seperately. Note that the order of values in each column must be the same so that the full set of data for a given entry can be reconstructed by querying the same index in each column file. 

### Column Compression
Column oriented storage is easy to compress. We can user bitmap encoding to reduce the number of bytes that are necessary to represent the required information. This method leverages the fact that in most cases the number of disctinct possible values is less than the total number of data entries. We take each distinct value and turn it into a binary variable, 1 if the given entry has the specified and 0 otherwise. If there are really a LOT of entries, we can further compress the data by run length encoding. This means that we count the number of contiguous zeroes or ones are present in the bit vector representing a given value. For an example, see below:

![[Pasted image 20220921201952.png]]

By using this compression method, we can make queries for large data very performant:

![[Pasted image 20220921202305.png]]
There are of course other approaches that aren't included here.

### Memory bandwidth and vectorized processing 
In cases of analytics performance, we have a bottleneck of getting data from the disk and into memory. That's what we have explicitly addressed thus far. However, there is actually additional benefits of using column storage with respect to making effective use of the CPU. Speificially, avoiding branch mispredictions and bubbles in cpu instruction pipeline and making use of single-instruction-multi-data instructions in new CPUs. 

By using columnar storage, we can be more efficient in our use of CPU cycles, this is because the query engine can take as input a greater number of data entries (since they're compressed) and iterate through them faster. The bitwise AND / OR operators are often designed to operature on such chunks of data. This is called vectorized processing. 

## Sort order in column storage
By default, the column storage architecture doesn't require any particular order for the data to be sorted, only that all the columns are sorted the same way since otherwise we won't be able to recreate a data record and different values will be jumbled up. Thus, the sort order becomes a business decision depending on the kinds of queries that we expect the database to be used for. For example, in the case of traditional business transactions, we might expect that we want to query by periods of sales. This means we may be interested in sorting first by date, making it easier to query a specific day, month or year of transactions. Within that, we may want to sort by product id, to easily calculate the number of sales of a given product withing a given time frame and so on. The first few sorting orders can be effectively compressed with run length encoding as above (since sorting means longer runs of the same value). Subsequent sorts will have fewer long runs of identifical values and so won't be compressed as much but still we're taking dubs regardless.

### Several different sort orders
What if we want different sort orders for different types of queries? Data replication is a standard practice to prevent loss of the data in case a machine fails. We might as well just store the data sorted differently and then use the version that best fits our query pattern. It's kind of like having multiple secondary indexes in row-oriented storage but without any references or pointers to data in other locations.

## Writing to column-oriented storage
So bare benefits to column oriented storage except actually writes are now more difficult. If we want to add some value in the middle of a column, we have to do so for every other column file so that the same order of values is preserved in all those columns. We can use LSM trees to fix this and the procedure is as follows:
- write to in memory store in a sorted structure
- after enough writes to the in memory store, merge with column files on disk
- write to the disk files in bulk

Since we wait for this aggregation of writes to the memory store, we need to make sure that we also query the in memory store when we do queries in the disk and then combind them both. We can hide this at the interface abstraction layer for the analyst user. Granted it's probably not necessary to even query the in-memory store a lot of the time.

## Aggregation: Data Cubes and Materialized Views
We often want to take some aggregate measure of the data n our data warehouse, such as sum, average or count in SQL. We can save computation by caching results for some of the most used queries and update them in place when more data is introduced. One such example is a materialized view. This is a virtual view that represents shortcuts to written queries rather than static, on disk written data. A common special case is the data cube / OLAP cube: 
![[Pasted image 20220921210632.png]]

We can see here that we have several aggregates precomputed for sale totals on specific days and/or for specific products. In practice, there can be many more dimensions that just two as this example demonstrates, but the approach can be scaled to N dimensions. The advantage is that these common queries are effectively precomputed and thus don't need to be re-written, values don't have to be re-scanned, etc... but of course for specific / special use cases, the data cube is not as extensible as a raw data query. The approach is typically to have raw data querying availble with a data cube supporting only the most frequent / standard queries to help with performance.  





