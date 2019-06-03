### Chapter 3: Storage and Retrieval
This chapter’s about how the internals of databases work. Why care? Because these internal choices determine the external tradeoffs any database has to make: between transactional vs analytic queries, read speed vs write speed, etc.

Two main categories: _log-structured_ storage engines and _page-oriented_ storage engines. (Log-structured ones are called that because they only permit appending to files on disk or deleting them.)

#### Data structures and indexes
Why not just store your data by writing into a file? You’d have to iterate over the whole file to read out particular values. In order to have acceptably fast reads, you need an _index_. The task of implementing an index consumes almost all of this chapter.

What is an index? Another data store derived from your primary data. Since it’s derived, it slows down writes, because you need to update the index when data is written every time. This is a key tradeoff in storage systems: the more indexes, the faster reads but slower writes. That’s why we don’t index everything by default.

#### Hash indexes
Say our primary data is being appended to a file. We could keep a hash map in memory from the primary key of a record to its byte offset in the file (i.e. its position). Every write would have to update that hash map. To allow us to use an append-only log as the underlying file, we treat updates as brand new records: write another line in the file, and update the offset in the index to point to the brand new line. This is the approach used by Bitcask, the default storage engine in Riak. 

What happens when the underlying file gets too large? Since it’s constantly being appended to, it’s going to strictly grow over time. One solution: close the file when it gets too large, and make future writes to a new file. In the background, compact the old files by throwing away duplicate keys 
(remember, every update makes the old log lines obsolete).

When the log file is split, we freeze the index and create a new one for the new log file. Queries that can’t find the record in the (initially empty) index will fall through to the index for the old log file, then the one before that, and so on). However, the old indexes can be merged in the background to keep the total number of segments small.

Implementation details: use a binary file format for the log, delete records by making a special update with a ‘tombstone’ special value, recover from crashes by storing snapshots of the index on disk, include checksums for the underlying file in case of corruption, and only allow one thread to write to the log.

Advantages:
* It’s fast, especially since many disk reads will be in the filesystem cache and thus also be in memory
* It’s suited if the value for each key is frequently updated but you don’t have too many distinct keys

Disadvantages:
* It requires that the whole index fit entirely in memory (the data store can be too large to fit in memory, since the values only live on disk).
* Range queries are not efficient, since you have to look up each key individually

#### SSTables and LSM-Trees
Suppose that we were able to sort the data in our underlying file by key. Wouldn’t that be nice? Merging files in the background would be easier, since they’d be already sorted. You wouldn’t need to index all the keys, only  a sample of them: if you wanted to look up the unindexed key “handsome”, you could start at the indexed key “handbag” and scan the underlying file from there until you find “handsome”. This is called a _sparse index_, and the underlying file is called a _sorted string table_ or SSTable.

How do you sort your data by key without giving up the performance benefits of an append-only log? You begin by storing your underlying data _in memory_ instead of on disk, at least at first, in a balanced tree data structure  (so that inserts in the sort order can be done efficiently). This in-memory tree is sometimes called a “memtable”. When your memtable gets bigger than a few MB, write it out to disk as a SSTable file - easy, because it’s already sorted - and point new writes at a brand new memtable. [I assume that the sparse index is generated when the memtable is written out to disk.] 

Read requests can check the memtable first, then fall back to the on disk segments from newest to oldest. Over time, old segment files can be merged and compacted. This is called a LSM-Tree (log structured merge tree): keeping a cascade of SSTables that are merged in the background. It performs well on large datasets.

Careful of _write amplification_ - when one write query causes 5x other writes as the db updates logs/on-disk files or compacts or merges later on. If the db is bottlenecking on disk write IO, write amplification can have a direct performance const. (This is unlikely on SSDs, but it’s still possible to wear them out more quickly by doing more writes)

This is basically what LevelDB/RocksDB/Cassandra use. (The terms memtable and LSM-Tree were invented by the Google Bigtable paper). 

Implementation details: Throw a Bloom filter in front of the db to prevent query misses from scanning through segment after segment. [Something about how and when to compact SSTables on disk - over my head].

#### B-Trees
At last we get to the most common type of index - B-trees. Forget the idea of storing data in segments that are kept first in-memory, then on-disk, and merged in the background. B-trees do it all on disk by breaking your data down into fixed-size blocks or _pages_ (kind of like how disks store data in blocks on the underlying hardware). Each page has a unique address that lets one page point to another.

The B-tree is a tree of pages. Each query starts at the root page, which contains alternating keys and addresses of child pages. Like a sparse index, the keys cover the whole data space sparsely. Each reference between two keys points to a page that is responsible for the space between those two keys, and contains its own more fine-grained keys, which bookend references to other pages, and so on. Eventually a query will reach a leaf page that contains individual keys and actual values.

Typically each page will point to several hundred child pages. Each page has a fixed size.

To write to a B-tree, you find the leaf page, change the value and write the page back to disk. No other pages need to be touched, either now or in a later compaction process. To add a new key, you find the leaf where it fits and either add it to the leaf page (if there’s room) or split it into two half-full pages. This ensures that a B-tree will never get unreasonably deep - mostly 3 or 4 levels in practice.

Since B-trees are updated in place (unlike LSM-trees, which just update an in-memory index [won’t the same read/writes problem still happen to the index in-memory?]), any implementation must protect the tree’s data structures from concurrent access with something like a mutex.

#### B-trees vs LSM-trees

General performance: LSM-trees are fast for writes while B-trees are fast for reads. Makes sense: LSM trees have to fall through multiple indexes for read queries, while B-trees have a standard < 3 level access path down the tree. And LSM-trees only need to update a memtable rb-tree for writes (not even write to disk), while B-trees have to search out the block on disk and overwrite the whole thing (or split it and write a new block in the worst-case), and write to the write-ahead-log for crash recovery.

LSM trees can be compressed better because B-trees have to leave empty space in each page (to accomodate new writes without splitting every time).

LSM trees are less predictable than B-trees because the compaction operation can interfere with ongoing queries - kind of like a garbage collection pause. Unlikely to impact average time much but can cause outlier queries to be slow.

LSM trees have a failure mode where compaction can’t keep up with writes and eventually the db runs out of disk space. Typical SSTable based storage engines don’t throttle writes automatically, so it needs to be handled at the operational level.

B-trees have only one instance of each key, while LSM trees can have multiple copies of a key in different segments. This makes it easier for B-trees to lock keys.

#### What values go in the tree?
What do you put as the value under the key in your B-tree or LSM-tree? It could be the whole db row (here the key would be a primary key or ID). This is known as a _clustered index_. Or it could be a pointer to a place in a giant heap file (though here it’s hard if the record grows substantially and has to be moved within the file - you either have to update all indexes or leave a forwarding pointer at the old location). Or it can be _some_ fields in the row, to make certain popular lookups easier, and a pointer to a heap file for everything else. This is known as a _covering index_, since the index itself covers some queries.

[Skipping some notes on multi-column indexes, coordinate indexes, and fuzzy search]

#### In-memory databases
Both B-trees and LSM-trees are ways to get around the limitations of disks. Memory is fast and reliable, while disks are slow and contain many other failure modes. We use disks because they’re cheaper, and because they retain data when turned off accidentally.

For data stores that don’t need to retain data, like caches, it makes sense to operate entirely in memory (see Memcached) and never write to disk.

If you do want to somehow retain data with an in-memory db, you need to (a) have special hardware, (b) write a log of changes to disk (cheating!), or (c) replicate your in-memory state to other machines [and, presumably, disks?] In-memory dbs must reload their state from disk or the network when they’re restarted. 

Examples: VoltDB, MemSQL, Oracle TimesTen, (sort of) Redis and Couchbase.

Why are in-memory dbs faster? _Not because they avoid disk access!_ In general, the OS caches recently used disk blocks in memory well enough for well-provisioned dbs to avoid being slowed down. They’re faster because they avoid the overhead of encoding and serializing their data structures in a form that can be written to disk. 
  [presumably they need to serialize for all _read_ queries, since an API can’t return a data structure. But they don’t need to store everything in a serialized state & then deserialize + serialize again for reads]

Since they don’t need to serialize [as much], in-memory dbs can store more complex data structures like queues and sets.

It’s even possible for an in-memory db to store more data than available RAM - the db can evict rarely-used data to disk and read it back in when it’s accessed, like the OS does with virtual memory/swap files. The index will still need to fit entirely in memory, though.  [Is this really an in-memory DB at that point?]

### Transaction Processing or Analytics?
Two very different models of db use: transactional or analytic. Transactional use generally operates on a small set of related records (e.g. a user’s purchases and buyer profile), and must be atomic (if you fail to purchase X, you need to cancel the whole order). Analytic use operates on a large set of unrelated records (e.g. all users’ purchase history) and can return partial results in the case of failure.

CRUD applications tend to be transaction processing systems, while analytics applications (like GA or Hive) tend to be analytic. The systems differ because you can’t easily optimize a db for both usage patterns at the same time. Latency and availability matter much more for the application than they do for the analyst.

Most large companies have a separate data warehouse for analytics. Some service periodically extracts data from the application-facing dbs and (after removing some sensitive fields) dumps it into the data warehouse. (Small companies actually can use the same db for both, because usage and data size aren’t large enough to matter).

[Skipping section on a “fact-table” schema for data warehousing]

#### Column-oriented storage
Transactional storage engines are designed for the whole record to be read back at the same time. If you care about a particular customer’s name, you likely care about their location/purchases/etc at the same time. In contrast, analytic storage engines usually only care about a small subset of columns across the whole data set (all the names and locations of all users, but nothing else).

To make these queries faster, the storage engine use a columnar layout: instead of storing all customer records in one (maybe segmented) file, store all customer _names_ in one file, _locations_ in another, etc. The nth row in one file will be the same customer as the nth row in another.

Columnar layout can make compression easier as well, because single-column data tends to be relatively homogenous. It’s easier to compress a list of numbers than a list of mixed strings, numbers and dates. 

[Skipping a section on sort order in columnar storage]

Writing to columnar storage is tricky because you can’t update a record in place without decompressing the whole column (and if it’s sorted, all the other column files, since the only identifier is position!) The solution is to use a LSM-tree: an in-memory sorted data structure that can receive new writes, then write them out in bulk periodically [which presumably does require decompressing a bunch of files]. Like a LSM-tree, read queries need to check the in-memory data as well as the disk data, but the query optimizer can do this in the background.

#### Materialized aggregates
Many analytic queries will be performing the same aggregates (counting or summing records). One way to cache these is a _materialized view_ - a table-like object whose contents are the results of some query. It differs from a virtual view because the data is actually query results written to disk, rather than a query waiting to be run. Every time the data changes, the materialized view needs to be updated (so be careful about adding too many!)
