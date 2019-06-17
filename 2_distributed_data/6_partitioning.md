### Chapter 6: Partitioning

If you have too much data to fit inside one node, you'll need to break up your data into partitions or shards. Each partition is effectively its own database, which shares a schema but no data with all the other partitions.

Partitions are normally replicated as well, so that it's less likely to lose a partition if a node goes down. Each physical machine (or datacenter) thus might have multiple partitions running on it, although most will be followers.

#### Replication and Key-Val data
How do you decide which records to store in which partitions? The goal is to spread the data and query load as evenly as possible, so scaling up by 10x nodes will let you store 10x the data and handle 10x the queries. (Data and load are not the same - some data might be requested much more than other data, like a celeb's tweets). When data or load is spread unevenly, this is called _skew_. A partition with disproportionately high load is called a _hot spot_.

The simplest approach: assign records to nodes randomly. The problem with this is that there's then no way of knowing which node an item lives on. In the worst case you'd have to send your query to every single node, all but one of which would respond 'not found'.

A better approach: pick an indexed key (like id) and partition by key range (like blocks in a B-tree). Think of an multi-volume encyclopaedia, where each volume is from (e.g.) "Bayou" to "Ceranothus". Now we know which partition a record is in, as long as we know its key and the key ranges in our partitioned db. You'd have to pick the ranges appropriately to avoid hot spot, and potentially change the key ranges as new keys are added. This is called _rebalancing_.

Careful of picking an indexed key that will naturally lead to hot spots! For instance, a timestamp is a bad key because all writes for a day will hit the same partition.

To avoid this risk entirely, many data stores use a hash function to select a partition. This way a lot of updates/reads of similar keys will still be distributed across all your partitions. No need to use a cryptographically strong hash, either, but you have to use a deterministic one (it can't depend on the process running the hash function, like Ruby's `Object#hash`!) This solves a lot of problems but does make it harder to do range queries, since adjacent keys will be scattered (by design) over all your partitions. One compromise here is to partition by one key but sort within each partition by another (e.g. partition by id and sort by timestamp). This way you still have to query each partition for all the records, but at least the range queries within each partition will be quicker.

Suppose you suddenly get a wave of writes on one particular key. Deciding where to store the record won't help you, since all requests are being routed to the same partition. Data systems generally can't automatically compensate for this (yet), but the application can, e.g. by splitting writes to that key over many randomly-closen sub-keys that are stored on different partitions, but are combined again by read queries.

#### Partitioning and Secondary Indexes
So far we've only talked about partitioning and indexing on one primary key. But real-world applications need to think about secondary indexes. How does a secondary index work when all its records are split across many partitions? Two main approaches here: _document-based_ and _term-based_.

Document-based partitioning requires each partition to maintain its own index that covers only the records stored in that partition: a _local index_. This makes writes simple, as each partition can easily update its own index. Reading is a bit trickier: you have to send the query to all partitions and combine the results. This is sometimes known as _scatter/gather_, because you're scattering the query to all partitions and gathering the results. It's an expensive operation.

Term-based partitioning requires us to maintain a _global index_ that covers data stored in all partitions. Note that you have to partition the index as well (e.g. the ids of red cars on one partition and the ids of yellow cars on another), but you can partition it in predictable ways that don't require querying all partitions. It's called term-based partitioning because the term we're looking for determines the partition (e.g. `color:red`). This makes reads easier but writes harder, because you might have to update indexes in multiple partitions (e.g. if the index `make:Audi` is on a different partition from `color:red`, for a red Audi.)

Updates to a global index are asynchronous, since otherwise you'd need a distributed transaction across many different partitions. [What are the consequences of this? Do records just go missing?]

#### Rebalancing Partitions
As things change over time, you'll inevitably need to move data from one physical node to another. Ideally the minimum necessary amount of data should be moved, without any downtime. How is this possible?

[Interlude about why we don't use the mod of the number of partitions to assign nodes to partitions: adding a partition would require moving most of the existing nodes around]

Ideally you'll move a partition at a time, not a record from one partition to another. This requires creating small partitions and assigning several partitions to each node. This way the number of partitions and the key-to-partition mapping doesn't change.

Why do we want to avoid changing the number of partitions? It's operationally easier [why?], but if you have to change it you can treat partitions like blocks in a B-tree: split large ones and merge small ones. It's also possible to make the number of partitions depend on the number of nodes: when a new node joins the cluster, have it split a fixed number of existing partitions and claim them [and then give them back when it leaves?] This is how Cassandra works.

Should rebalancing happen automatically or require manual intervention? (Note that even manual intervention can be system-assisted: Riak, for instance, suggests a partition assignment but requires an admin to manually approve it). Fully automatic rebalancing can be easier, but beware! It's a very expensive operation because it requires moving a ton of data over the network. It's easy to cause a cascading failure by moving data from an overloaded node, thus overloading the other nodes, etc etc.

#### Request Routing
Suppose you have a partitioned dataset, which is partitioned by some set rule. How does a client know which partition to send the request to? We can't rely on the initial key ranges (or whatever) never changing, as they might change as the partitions are rebalanced. This is an instance of the general _service discovery_ problem.

Three high-level approaches: allow clients to contact any node and have those nodes proxy the request along; make clients contact a routing tier first which proxies the request to the right node; or require clients to be aware of the partitioning before making any requests. In any approach, the same problem exists - how does the component making the routing decision know which record lives where?

This is a problem of _distributed consensus_. Most data systems use a separate service designed to do distributed consensus right, like ZooKeeper. Some (like Cassandra and Riak) use a gossip protocol between nodes instead, and have each node proxy requests along.

[The chapter hints at much more complexity in partitioned analytic datastores which will need to run many large queries across partitions, to be discussed further in chapter 10]
