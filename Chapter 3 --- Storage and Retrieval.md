
## Logs and hash maps

* Strictly speaking, a **log** is any append-only sequence of records. Logs may be human-readable, or they may only be machine-readable.
* This is distinct from application logs, which are a specific type of log. The two concepts are sometimes conflated however.
* Logs are a fundamental structure in a database. Logs are fast because it's hard to beat the performance of simply appending to a file.
* Another fundamental structure is the **index**. An index is a data structure which speeds up read operations. It usually slows down writes however because it requires performing work on every insert or append operation.


* The **hash map** is a simple but effective index structure.
* Both Python and JavaScript objects are ultimately just implementations of hash maps. So are the `dict` and `Map` classes in these languages.
* Databases using logs for primary storage are said to be using **log-structured storage**.
* If you combine a log and a hash map, you have a database! Thus the simplest possible practical (you can technically omit the index, but then a read operation would require a full file scan!) database: an append-only file store with an in-memory hash map index pointing to the byte offsets of records of interest.
* [Bitcask](https://en.wikipedia.org/wiki/Bitcask) is basically this, and it's a backing store for a number of more advanced architectures, as well as a database option in and of itself.
* This architecture has lots of advantages:
  * Low read and write latency, with predictable timing. Reading is just a single disc read, writing is a single disc write.
  * High throughput.
  * Backup is as easy as copying the file.
  * Straightforward error recovery. Only the last element being written can get corrupted.

* Of course, it has disadvantages as well:
  * The index must fit in memory (otherwise performance plummets).
  * Range queries get no speed-up (which e.g. time-series data would benefit from).
 
 
* If log-files are left untouched, they will grow to infinity in size.
* In practice you can split your log into *segments* of a certain size, and then periodically **merge** and **compact** those segments into a merged log.
* The compactor iterates through the log file and, for every key written to, finds the most recent value written. These operations are kept in the log, and as part of the merge all previous writes are discarded.


* There are some additional considerations when implementing log files.
* You will want to **compress** the log using a binary encoding scheme.
* A special value called a **tombstone** needs to be appended to the log in the case of key deletion.
* You can recover from a volatile memory crash in the hash map by reading from the start of the log segments, but this can take a long time if the segments are large. You can speed this up by keeping a snapshot of the hash map backed up to the disc.
* You can recover from a database write crash by calculating a checksum on the data segment being written ahead of time, and making sure, when recovering, that the check of the data included in the last write matches your checksum.
* Log files are sequential append-only, so they support multiple reads, and hence concurrent requests, if you want to service that.


* If we insert data into the log in sorted order we can maintain a much smaller log by overwriting prior value assignments in-place. Why don't we do that?
* Mainly because insertion in the middle of a file is a **random write** operation, while appending to the end of the file is a **sequential write** operation.
* In a random write we seek-write-seek-write.
* In a sequential write we seek once, then write multiple times (append from the same cursor).
* The latter is always faster than the former as disc seeks are extremely expensive. Since writing to the log has to happen every single transaction, this time difference matters a lot, and makes log-writing the more attractive option.
* Additionally, the simplicity of the log file makes concurrency and crash recovery much easier to implement.

## SSTables and LSM-trees

* The **SSTable** (Sorted String Table) is a more complex log-structured storage implementation.
* In SSTables we still have log segments, but their contents are now sorted by key.
* This design employs a balanced in-memory data structure, in this context referred to as the **memtable**. The memtable needs to retain key sort order.
* Every time a write or modify operation occurs, the data is written to the memtable.
* The memtable is periodically flushed in key order into a new log segment. This is still a sequential write!
* Those segments are periodically compacted and merged. The difference is that since the segment keys are in sorted order, this process is greatly expedited: we no longer need to scan through every value in every segment, as we can skip over stale key modifications completely!
* Because we no longer write to the log right away, and instead buffer using an in-memory data structure, we must keep an additional unordered log file in play, one that gets written to whenever the memtable gets written to. This is so that the memtable can be repopulated in the event of a crash.


* One advantage of this design is, as previously mentioned, that compacting segments is now much more efficient.
* Another advantage is that you no longer retain a complete index in memory. So this design works with large databases whose hash table index wouldn't fit in memory.
* If all data values were the same size, we could look up values by performing a binary search. However, in practice they are not.
* A sparse in-memory index is retained. At read time this index is queried for the range of memory values a database element could be located at, by finding the immediately smaller and larger keys in the index and taking the range of values in between their endpoints. That range is then scanned to find the value.
* For maximum read efficiency, keep the ranges down to the hardware page size.
* The process of actually scanning the data, once you have the pointer, is extremely fast.
* One more practical modification comes from the observation that with current hardware, disc I/O is more precious than CPU cycles.
* So most databases compress the index address space reference blocks. This makes the data smaller and leads to less time spent reading from persistent storage to volatile memory, but also some CPU cycles expended on decompression.


* Detour note: the memtable in the SSTables implementation needs to be a data structure that affords random insertion order and sorted read order. There are tree structures that have good performance for these tasks, namely red-black trees and AVL trees. I implement these myself in other notebooks in this repo.


* This arrangement is what LevelDB and RocksDB, among other databases, use.
* Cassandra and HBase use a similar scheme derived from the Google Bigtable paper.
* Lucerne, a full-text search indexing engine used by Elasticsearch and Solr, uses a similar method for storing its (much more complicated) term dictionary.


* For sufficiently small amounts of data, a log-structured hash table database is faster.
* But SSTables has several advantages. The most important ones being much larger practical memory limits, due to the sparse in-memory structure, and intrinsic support for range queries.
* There are many potential performance tuning measures that can be taken.
* For example, the SSTables architecture can be slow when looking up keys that do not exist in the database, or which have not been written to in a long time, because walking back the memtable and the list of SSTables takes time.
* You can use a **Bloom filter** (note: implemented in another notebook) to speed this process up. This algorithm is great for approximating set contents.


## B-tree

* The most common database implementation, and the one used by most of the \"classic\" SQL engines, isn't a log storage design, it's a **B-tree** design.
* The B-tree is a data structure which provides balanced in-order key access. It's not dissimilar to the red-black trees, actually.
* Note: implemented elsewhere.
* B-trees are organized in terms of **pages**. Each page contains references to pages further down the list, except for the last page (the leaf page), which inlines pointers to the data it references.
* Pages are traditionally 4 KB in size, and are designed to emulate the way that memory pages in the underlying hardware work (keeping B-tree pages smaller than hardware pages speeds up access!)
* To look up a value, you start at the root page, whose keys describe a range of values indexed by sort order. You find the key-value pair associated with the index range your value is in. The value will be a pointer to another page, with another range of valid values. Keep burrowing until you get to a leaf page, which contains an exact index location key and a pointer to your desired value. Tada!
* To insert a value, find the legal space for it amongst the leaf pages. If any of the pages along the way are too full to accept new values, split them into half pages, and refactor references appropriately.
* Deleting a value is a lot more involved, however.
* B-trees are balanced trees: for $n$ keys they have a height of $O(\\log{n})$.
* B-trees operations are not intrinsically safe. To make them safe you need to add a **write-ahead log**, and write the B-tree operations that are necessary to that log before performing the actual operations.
* In contrast to the file append-only operation of the log-structured database, B-tree databases need to do seek-writes to specific memory pages, which is slower. On the other hand, they do not need to perform the background write operations necessary in log-structured databases.
* Additionally, if you allow for concurrent access (and concurrent modification), you need to introduce locks (specifically, latches on the level of the tree being modified) in order to prevent ongoing writes from causing ongoing reads to return inconsistent data. SSTable designs are much less complex in this regard.
* Some B-tree optimizations:
    * Instead of overwriting pages and maintaining a write-ahead log you can use a copy-on-write scheme: write a new page with the included modification elsewhere in memory, atomically swap the reference from the old page to the new page, and delete the old page. This improves concurrenct read performance, at the cost of write performance. LMDB is an example of a database that implements this.
    * In higher-level pages, instead of storing entire keys store the leading bytes of the keys. Keys only need to provide enough information to boundary where to go next, and smaller keys allow fewer more packed memory pages, a higher branching factor, and ultimately shallower value access. This is known as the **B+ tree**.
    * B-trees try to store their leaf memory pages in sequential order, obviating the need to seek (for certain access, and only when the hardware complies).
    * Pointers may be added to leaf pages pointing to their immediate left and right companion leaf pages. This increases range query speed as it obviates the need to return to the parent page.
    * Finally there are more complex B-tree variants, like **fractal trees**, which attempt to further reduce seek volume.
    
    
* Comparing B-trees and LSM-trees...
* B-trees involve seeks, so they're slower when read volume is high. This is the primary thing that makes LSM-trees more attractive.
* On the other hand, they do not involve periodic compaction. Compaction uses slow disc I/O resources. It's possible to schedule it when nothing else is hitting the disc, but hard to do so reliably. This means that, particularly at higher percentiles, B-trees have more reliable performance than LSM-trees do.
* In general it's hard to predict which of these two structures will perform better on a particular workload. You have to test emperically.

## More index considerations
* Secondary indices are barely different from primary indices. The only implementation differene is that they allow duplicate keys, so you need to bake a slightly different key. An easy way around this is to append the row number.


* The key in a primary index is always an identifier for the thing you are searching for.
* The value however could be one of two things. Either it's exactly the data in question, or a pointer to that data somewhere else in memory.
* The latter is the **heap file** approach.
* The heap file approach is more common. If multiple indices are defined on a particular chunk of data, it avoids duplicating it.
* An implementation detail is what to do when inserting new data.
* If the new data is smaller than or the same size as the previous data (as it would be when you have a fixed-width type!), you can overwrite in place.
* If the new data is bigger, you need to either write to a new location and move the reference, or write to a new location and populate a pass-through pointer.
* The jump to the heap location is a seek, so it has a time cost. Storing the data directly in the index instead is known as a **clustered index** approach, and it avoids this cost.
* Secondary indices in a table with a clustered index \"just\" populate references to the primary index.
* However, a clustered index represents duplication of data, since obviously that data has to exist in the log first. Thus you trade worse write performance for better read performance.
* Also, it hits size limits much faster, if your objects are large.
* A compromise between the two is a **covering index**, which stores just *some* of the columns in the index. Write operations on these columns are slower, but read operations are faster. Columns not \"covered\" are faster.


* Finally, there are multi-column indices. These speed up queries that pivot on several columns at once.
* The most common type of multi-column index is the **concatenated index**, which is keyed using a sequence of values. The semantics are slightly different, but the structure is mostly the same.


* There are more complex index structures, like R-trees and whatever the text search engines use. But they're slightly out of score for now! Something to investigate further down the line.

## OLAP systems
* Databases in production can generally be split into two access patterns. **OLTP** systems (short for online transaction processing) is the original use-case. **OLAP**, short for online analytics processing, is the new use case.
* The operational reasons for splitting OLTP from OLAP I know too well!
* OLAP systems are less creative than OLTP systems in terms of overall design. Almost all applications use some variant of the **star schema**.
* In the star schema there is a central table that acts as an access point, which provides foreign keys to a bunch of other tables.
* Each of the other tables is known as a **fact table**, and it provides details about one specific aspect of the elements in the central table: who, what, where, when, how, why?
* A more complex and normalized variant of this schema is the **snowflake schema**, which has additional branching factors on the fact table.
* These arrangements are so named because they look a star or a snowflake surrounding a central fact table in the middle.


* OLAP systems injest data every once in a while. Simultaneously, the queries that are run against them are often very heavy, touching hundreds of thousands and potentially billions of rows.
* One way to increase query efficiency is to maintain a **materialized view** of oft-used summary statistics. This is known as a **data cube**.
* In a data cube you choose a certain number of dimensions $n$ (an $n$-fold data cube), and then compute a summary statistic against the resultant aggregate.
* If those statistics are used often, it may be worth paying that cost up-front once at injestion time, instead of having to rebuild every time a query using that summary comes in.


## Column-oriented storage

* While the OLTP access pattern is row oriented, the OLAP access pattern is column-oriented. For efficient OLAP architecture, you want to provide locality on the columns, *not* the rows.
* Thus the natural OLAP adaptation: **column-oriented stores**.
* Column-oriented databases fundamentally lay colums contiguously in memory, instead of rows.


* Additionally, ordering data in terms of columns unlocks columnar compression.
* There are various schems that you can use. One of the most common is the **bitmap encoding**, which calculates a dummy encoding for each product and stores it.
* When there is low columnar cardinality, bitmaps are very efficient memory-wise. They are also essentially **vectorized** already, so math operations (like selection with `OR` clauses) are extremely fast, making common queries much faster
* You can compress further using something along the lines of **run-length encoding**.
* Compression is particularly important in OLAP systems because these may need to scan millions of entries at a time. Sequential disk I/O is the bottleneck, not entry look-ups, so reducing the memory footprint is crucial.
* Compression is a whole other topic worth exploring.


* You still insert data into column stores in a row-wise manner.
* It probably doesn't make sense to use B-trees to store the data, as overwriting data with new values that are too large to fit in the allotted memory space will break column contiguity.
* Log structured storage still works well, however. (need to think some more about why this is the case)


* When data is organized in column order, there is no intrinsic row sort order.
* Leaving the rows unsorted improves write performance, as it results in a simple file append.
* Sorting the rows improves read performance when you *do* need to query specific rows in a column-oriented database. You can multi-sort by as many rows as desired, obviously, but rows beyond the first will only help when performing grouped queries.
* Additionally, sorting keys can help greatly with compression. Especially on the first sort-order key, something like run-length encoding can result in incredible read performance. With low enough cardinality multiple gigabytes of data can get pushed down to mere kilobytes in size!
* A clever idea is to actually maintain multiple sort orders on disc, by replicating the data with several different sorts. Obviously this is taking read performance at the cost of write performance as far is will go. Vertica is one example of a database that offers this feature."
