### Chapter 5: Replication

Replication means keeping copies of the same data on multiple machines connected by a network. This chapter assumes that your whole dataset fits on a single machine (see chapter 6 for what happens when it doesn't). If the data you're replicating will never change, replication is trivial: just copy the data. The difficulty is in handling changes to replicated data, since those changes must themselves be replicated and kept in-sync across the network. There are three popular ways of supporting this: single-leader, multi-leader and leaderless.

#### Leaders and followers
Call each node that stores a copy of your data a _replica_. Each new write needs to be processed by every replica. One easy way to do this: designate one replica as the _leader_ (or 'master', or 'primary), and point all client writes to this replica. When this replica gets a write, it handles sending that write to all other replicas, which apply the write to themselves. That list of writes is called a _replication log_. Clients can read from any replica, but must write only to the reader. [Presumably this is "single-leader"]

Examples: Postgres, MySQL, Mongo, etc etc.

### Synchronous vs asynchronous replication
Does the leader wait for an OK from its followers before it responds OK to the client? If not, then the system can't guarantee that data is kept if the leader suddenly fails. If it does, then the system risks high latency on writes as the leader waits for all its followers. Many systems compromise by nominating a single follower as synchronous and keeping the rest async, thus guaranteeing an up-to-date copy of the data on at least two nodes. But it's also okay to go totally async, as long as you can accept the possibility of losing writes.

### Setting up new followers
How do you stand up a new replica and ensure it has an accurate copy of the leader's data? You can't just copy files, because the data will have changed by the time the copy finishes. You could lock writes while you copy the data, but that would cause downtime. Instead, copy files in the background, then have the follower request all the changes that happened since the data was copied. (This means you need a way to identify an exact location in the leader's replication log.) Only have the follower accept reads once it's caught up on these changes.

### Handling node outages
Any node can die at any time. When a follower replica dies and comes back, it's easy to recover: request a list of the changes that happened while it was dead, then apply them like a new replica being stood up would. When the leader dies, it's trickier. A follower must be promoted to leader, and clients/other followers must point at the new leader instead of the old one. This is called _failover_.

Automatic failover requires automated ways to determine that the leader has failed (e.g. a healthcheck), to choose a new leader (generally the most up-to-date follower), and to point the system at the new leader (including isolating the old leader if it pops back up). This is a fraught process:

* The new leader might not have some of the recent writes, which may become available if the old leader pops back up. What happens to those out-of-sync writes?
* Other systems depending on db contents might respond _very_ badly to losing writes. For instance, if the db is assigning autoincrementing primary keys that are then stored in Redis, failover might result in duplicate keys and data leakage to the wrong users!
* It's possible for two nodes to both believe they are the leader, which could result in your data splitting into two copies that are out-of-sync and cannot be safely merged. If a leader is configured to STONITH when it sees another leader, both leaders could kill the other.
* It's hard to determine the right period to wait before deciding a leader is dead

These issues: node failures, unreliable networks, and tradeoffs around consistency/durability/availability/latency are fundamental problems in distributed systems.

### Implementation of replication logs
How precisely does a leader send its queries to followers?

The simplest solution is _statement based replication_: keep a log of every query it recieves (`SELECT * FROM...`) and send that log to followers, which parse those queries as if they'd been recieved from a client. However, this won't work for nondeterministic functions like `NOW()` or `RAND()`, since those will be different between replicas. Some statements will also depend on existing data (`UPDATE * WHERE...`), which means each statement must be executed in exactly the same order, which is tricky when there are multiple concurrent transactions [why?]. Lastly, statements with side effects (triggers, stored proceduers, etc) may result in different side effects on each replica. This was used in MySQL before v5.1 - with some workarounds for these issues - but most data stores use a different strategy.

Another solution is _write ahead log shipping_. Remember how storage engines append writes to logs (either as the main heap file in a LSM-tree, or as the WAL in a B-tree)? We can ship that log directly to each follower. This is used in Postgres and Oracle. One problem is that the WAL describes which bytes were changed in which disk blocks, which tightly couples the log to the storage engine. This means you can't run different versions of your db on leaders/followers, which means you can't run rolling upgrades of the db in a cluster. This is why db engine upgrades often require downtime.

A third option is _row-based log replication_. This is like using a WAL, but in a separate, more general encoding that won't change as the storage engine changes. It tracks changes on the column level, not on the bytes-in-a-file level. One advantage of this strategy is that it makes parsing the log easier if you want to send it to a warehouse or build a custom index.

A final option is _trigger-based replication_. If you need very application-specific replication logic, you can do all the replication work in a database trigger rather than relying on the db itself to do it. This is obviously much much slower and buggier, but sometimes necessary if you need to be really flexible.

### Problems with replication lag
Suppose you have some asynchronous followers in your replicated data store. Reads from those followers might not capture the most recent state of the data. The data is only _eventually consistent_, not immediately consistent. The delay between a write happening on a leader and being reflected in a follower is called the _replication lag_. It's usually only a fraction of a second, but in many failure modes it can increase to seconds or minutes.

#### Reading your writes
One big problem with replication lag is "reading your own writes". A common application pattern is to write to the db (submitting a comment, for instance) and then immediately read out the changed data (displaying the new list of comments). Since reads go to a follower node that might be out of sync, if the application submits its read request quickly enough it can get a snapshot of the data before its own write happened. To the user, it will look like their comment was not saved.

Being able to read your own writes is called "read-after-write consistency". Note that it only guarantees that users can read _their own_ writes. Other users' updates may not be immediately visible. There are a few ways to implement read-after-write consistency:

* When reading something the user might have modified, read from the leader instead of a follower
* If updates are rare, you can read from the leader for a set period of time after an update (e.g. a minute)
* A client can send a timestamp of its most recent write and send it along with read queries. The system can then ensure that a read only gets served by a replica that is up to date with that timestamp

Note that this gets harder if your replicas are distributed across multiple datacenters, or if you're trying to get read-after-write consistency across multiple devices (e.g. a user's phone and their computer).

#### Monotonic reads
Another problem with replication lag is that it's possible for users to see data moving backwards in time. If a user makes multiple reads from different replicas, later reads might hit more out-of-sync replicas, causing the user to see (e.g.) new comments disappear. You can get around this by sending a timestamp with reads or by ensuring users read from the same replica.

#### Consistent prefix reads
If the database does not guarantee that writes will be applied in the same order, different replicas may apply those writes in a different order. To someone watching the data, this can lead to apparent violations of causality. If table B is being updated in response to the changes to table A, observers may see the update to B happen before the update to A. Keeping track of these dependencies can be difficult.

### Multi-leader replication

If you only have one leader, then that's a single point of failure for your data store. What if you could accept writes on multiple nodes, in a _multi-leader_ configuration?

Generally not useful in a single datacenter. You only need the complexity if you you want a leader in each datacenter. Having a leader in each datacenter gives you significantly better performance and tolerance of outages/network problems. Each leader will replicate its changes to the leaders in all other datacenters, as well as to its followers in its own datacenter.

The obvious drawback: if the same data is modified by two different leaders, how do you resolve the conflict?

Other than distributed databases, there are two very different use cases here: offline app functionality (e.g. calendar sync), which needs to support reads and writes offline and then reconcile them when the app goes back online; and real-time collaborative editing where each editing user effectively acts as their own leader node. Both of these need conflict resolution.

#### Handling write conflicts

Could you wait for each write to be replicated to all other leaders before reporting a successful write? Yes, but the latency and durability tradeoff will make your setup strictly worse than a single-leader setup. No point in doing multi-leader if you don't allow each leader to accept writes independently.

The neatest way is to ensure that write conflicts never happen: e.g. by ensuring all writes to a particular record go through the same leader. For instance, in an app where users only edit their own records, each user could be assigned a datacenter/leader. This works until a datacenter fails and you have to send writes to a different leader.

If you can't do that, you need a way to enforce a strict global ordering of writes, so they can be applied in the right order. That way even if a conflict causes one write to fail, at least every datacenter will maintain the same state. You could do this by assigning unique sortable ids to each write, or each replica.

If you can't support the possibility of losing writes, you need to store the conflict for later resolution: either by concatenating the conflicting values, or by storing them in a data structure that preserves all the information. [Does this get increasingly complicated as you keep writing to that value?]

Since appropriate conflict resolution will probably depend on the application, most databases that support multi-leader replication will also support custom conflict resolution logic. You can run this either when a conflicting write is detected, or when the conflicting data is read (here you need to store the conflicts in the meantime).

#### Replication topologies

How does each leader replicate to each other leader? You could send all the writes to all other leaders in a web, or have a limited set of leaders you communicate to. MySQL by default only supports a _circular topology_ where each leader listens to one leader and speaks to another. This means less chatter but also requires tagging writes with unique IDs so nodes don't endlessly forward and apply them in a circle. Circular topologies can also break replication if one node in the circle dies.

Why not always use all-to-all topologies? If there's no single order in which writes propagate, some writes can overtake other writes on some replicas. Leader A could create a record which leader B then updates, but leader C could receive the update query before the create query. Since clocks in distributed systems cannot be trusted, you can't rely on timestamps to order this properly. [Unique sortable ids on each write won't solve this either, because they only guarantee a single order, not the right order?] 

The take-away from all this? Multi-leader replication is _really hard_ and don't trust your database to do it right! Lots of tricky failure modes.

### Leaderless replication

What if we just gave up on trying to correctly order writes and let every replica accept writes from clients? Here a replica will mark a write as successful if it gets a _quorum_ of OK responses from other replicas. This means some replicas might miss some writes (in the case of network failure). To mitigate this, read queries must be sent to several nodes in parallel, so the application can display the most recent value.

Dynamo is an example of this.

How do nodes catch up on writes they missed, with no single source of truth (the leader's replication log) to rely on? Two mechanisms: _read repair_, which means that when clients make read queries to multiple replicas they update the ones that are out of date, and _anti-entropy_, where some background process polls all the replicas and catches them up to each other eventually. Relying on read repair, like some systems do, risks rarely-read values being missing from some replicas for a long time.

What's a sensible number to set as a quorum? If we have n nodes, we need our read quorum plus our write quorum to be greater than n. Otherwise we could have a successful write followed by a successful read that does not contain the written data. (But if you don't strictly need this, it is possible to configure the quorums differently!) Within this constraint, you can manage your quorums to support your specific use cases. If you have few writes and many reads, you want a large write quorum and a tiny read quorum to make reads faster and more reliable/lighter weight. However, any small quorum risks serving or writing bad data if all the nodes in your quorum fail.

[There are lots of interesting tradeoffs in how you set your quorum numbers here that I'm glossing over.]

#### Sloppy quorums and hinted handoff

[This section seems to presume a sharded database when it talks about "nodes where the value usually lives"]

If a write request can get a quorum of nodes, but not a quorum among the nodes which host a particular value, some databases will accept that write and set the value on nodes where it doesn't usually live. This is called a _sloppy quorum_. Once the network interruption is fixed, writes like these are sent on to the nodes where they belong. This is called _hinted handoff_.

#### Detecting concurrent writes

In a datastore that allows multiple leaders to write the same value, how do you resolve that conflict? [This expands on the "strict global ordering" bit in "Handling write conflicts" above].

The simplest approach is probably _last write wins_. This approach requires a strict global ordering of writes. The write with the most 'recent' place in the order wins (even if it wasn't temporally most recent). Obviously this risks losing writes that get trampled by more recent writes.

What does "concurrent" mean? How do we decide what happens before what? We want to cover cases where a value is created and later updated - the update causally depends on the create. An operation A happens before B if B depends on A in some way. Two operations are concurrent if neither knows about or depends upon the other. Thus algorithms that determine concurrency are really about _dependence_, not about time.

[Skipping a discussion of a particular algorithm for determining which writes are concurrent. It sounds a lot like storing conflicts and relying on clients to resolve them at read time - see the last point in "Handling write conflicts" above. In short: each write gets a version number that's used to differentiate updates from concurrent writes.]
