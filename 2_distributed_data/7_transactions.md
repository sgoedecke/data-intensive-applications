### Chapter 7: Transactions

Lots of things can go wrong with data systems: hardware or software failures mid-write, network partitions, concurrent writes overwriting each other, reads of partially-updated data, and other race conditions. Transactions are a way to be fault-tolerant with respect to these risks. Transactions group a block of reads and writes into one atomic operation that either succeed together or fail and rollback. By making partial failure safe, we guard against innumerable things that can cause partial failure.

### Transactions and ACID
Transactions have been effectively the same across decades of competing relational db implementations. NoSQL databases, by contrast, have abandoned or significantly weakened transactions. The marketing buzz around these claims (falsely, according to the book) that transactions are inherently unscalable. In reality there are a bunch of tradeoffs.

The specific safety guarantees of transactions are often called _ACID_: atomicity, consistency, isolation and durability. Not every db agrees on the definition of these!

Atomicity means that queries are grouped into an atomic - indivisible - unit. If one query fails, all writes from the unit are discarded. "Abortability" might be a better term.

Consistency means a whole bunch of things in data systems: eventual consistency, linearizability, consistent hashing... In ACID, consistency means that your data is internally consistent: e.g. that credits and debits are balanced in a banking application. This isn't something the database can guarantee, so it shouldn't be in this acronym [Really?].

Isolation means that concurrently execuring transactions cannot interfere with each other. For instance, if a counter is at 1 and two transactions attempt to increment it, it should end up as 3 instead of 2. Very strong isolation is "serializability" - the idea that concurrent transactions should leave the data in the same state as if they'd run consecutively - but few dbs guarantee this.

Durability means that successful writes will not be lost if the database crashes or the power goes out. This is usually a matter of persistent storage/replication and can't be fully guaranteed.

### Multi-Object Operations
Atomicity and isolation become very important when multiple objects need to be kept in sync. For instance, if a counter of unread emails is stored in the db (as a denormalized cache), partial failure might lead to the user seeing conflicting information.

[Rest of the chapter omitted for now]
