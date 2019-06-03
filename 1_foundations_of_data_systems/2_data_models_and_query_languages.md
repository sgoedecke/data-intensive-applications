### Chapter 2: Data Models and Query languages

The two main approaches: the relational model (e.g. SQL), and the document model (e.g. Mongo).

Object-oriented application code must interact with a relational data store through an ORM like ActiveRecord. Some think this is an awkward mismatch and using a JSON model is more natural (because when you do `post.comments`, the comments are right there in the JSON but you have to look them up in another table if you’re using a relational store).

Why use IDs in a data store? Removing duplicated data - _normalization_ - has all kinds of benefits: easier to update, less ambiguous, more consistency, better search… Normalization usually requires a many-to-one relationship which is easy to do in a relational store but difficult in a document model.

#### The network model
The network model is an ancient idea that is pretty similar to the document model. Like the document model, the data is represented as a graph with parents and children. Unlike the document model, each record may have multiple parents. It’s more like a network than a tree.

The only way to access a record is to follow a path from a root record along a bunch of links - an _access path_. Often several possible paths can lead to the same record, making the application programmer’s job difficult.

#### The relational model
The relational model lays out all the data as a bunch of rows in tables with no explicit links. When you write a query against this data, the query optimizer decides which parts to execute first and what indexes to use - effectively pre-computing the access path that programmers had to manually worry about in the network model. The big win here: write one complex, difficult query optimizer, and it saves hundreds of thousands of hours for every other developer actually writing queries. (Kind of like writing a really smart compiler.)

#### The document model
The document model is actually more like the relational model than the network model. It lays out all the records next to each other (not in a network) and uses _document references_ (i.e. foreign keys) to link many-to-many or many-to-one relationships.

Limitations: 
* Poor support for joins
* Hard to refer directly to a nested item within the document (you have to effectively specify an access path like `record.children[2].children[3]`

Benefits: 
* It’s simpler and faster for data which has a document-like structure already (a tree of relationships where you load the whole tree at once).

So which is better? Well, it depends on your data and how you expect it to be queried. This is a refrain you’ll hear a lot in this book!

#### Schema flexibility and data locality
Relational databases enforce a strict schema on their contents. Document databases don’t - you can add arbitrary keys and values. This doesn’t quite make them _schemaless_, because the application code usually has to do the job of enforcing some kind of structure. The book recommends calling document databases “schema on read”, and relational databases “schema on write”. It’s like runtime vs compile-time type checking.

What happens when you need to change the schema? In a document db, the application code has to remember the old schema for all time (!) and automatically translate for old records. [Surely that can’t be right!] In a relational db, you run a _migration_ that alters the table. Runs in a few ms on most relational db systems, but can be orders-of-magnitude slower on MySQL.

Why use schema-on-read? Useful if your data is highly heterogeneous, or if you have no control over the structure of your data. Other than that it’s probably better to enforce schema-on-write.

In a document db you need to load the whole document from the same place for reads or writes. This can be a benefit (if you need to load it all anyway, it’s faster to do it like this than go scavenging across tables) and a disadvantage (if you only need a small portion, you end up loading/rewriting a lot of wasted data).

#### Common ground?

With relational dbs supporting JSON or XML storage, and document dbs beginning to support relational-like joins, the book suggests that the relational and document models are beginning to merge a little bit, which makes sense: if you can store the relational-like bits of your data in a relational db, and the document-like bits of your data in a document db, it’s convenient for them to be the same db!

#### Query languages for Data
Remember the network model? Queries had to be imperative, since they specified a particular access path. “Look here, then there, then there, then return that record”. In a relational model, queries can be _declarative_. `SELECT * FROM animals WHERE family = ‘sharks’` doesn’t tell the db how to get the data - the query optimizer handles all of that. This means (a) that you don’t have to do the work, (b) that the query optimizer can improve over time and you’ll automatically reap the benefits, and (c) that the work can be more easily parallelized.

Declarative query languages are nicer in lots of places. Compare the ease of styling in CSS compared to having to manually set the style attributes in JS.

#### MapReduce
MapReduce is kind of a hybrid between imperative and declarative query languages. You have to write imperative `map()` and `reduce()` functions, but the underlying system handles parallelizing calls to those functions and wiring them up to return you a set of results. MongoDB is an example of a data store that supports this kind of query.

Disadvantages: it’s more difficult to write correct queries and the database is limited in how it can optimize.
Advantages: you have a lot more options about what you want to do with your queries, since you have a whole language at your disposal.

MongoDB also supports expressing your map and reduce steps more declaratively, as a raw object instead of a pair of functions. The book suggests that this is effectively re-inventing SQL.

#### Graph-like Data Models
If essentially all your data is in a many-to-many relationship, it becomes more natural to model your data as a graph: for instance, points in a road network, or websites linked by HTML links.

You could represent a graph store in a relational db by having a vertices table and an edges table, with a many-to-many relationship. Any vertex can be connected to any other vertex, and that connection - that edge - can be labeled according to its relationship.

[Skipping a lengthy discussion of different types of graph query languages: SPARQL, the semantic web, the RDF data model. The short version: they support declarative queries, unlike the old-style network dbs, they support accessing vertices/edges by unique id, queries tend to be nicely composable.]

