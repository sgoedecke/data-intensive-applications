### Chapter 4: Encoding and Evolution
The last chapter was about ways to quickly read and write data to disk. This chapter's about what you actually do when you're writing data to disk. In memory, data is represented in a structured way, but on disk it's just a stream of bytes. Any application that writes data to disk needes a way to encode structured data into bytes and decode it later.

Once you store data to disk, it won't change. Any serious application will need to alter its data schema at some point. How do you keep old data accessible when your application transitions like this?

#### Formats for Encoding Data
Many languages support ways to directly jam a data structure (e.g. a Python class) into a byte sequence (Ruby's `Marshal`, Python's `pickle`, etc). This is convenient but (a) commits users of that data to the programming language that encoded it, (b) means the decoder must be able to instantiate arbitrary classes, which is a big security problem, (c) contains no way to version data if the schema does chnage, and (d) requires a lot of CPU to encode and decode. Don't use them.

More useful are standardized encodings that can be read by different languages: most popular are JSON and XML. Some problems with these: (a) number encoding is tricky, in particular distinguishing numbers from strings or floats from ints, (b) encoding binary strings is tricky without base64 encoding them, (c) enforcing a schema must happen at the application level. In the book's view, getting people to agree on any encoding is the hard part. These problems are trivial in comparison.

If you want a smaller payload that's faster to parse, binary encoding is a good choice. It's possible to directly binary encode JSON, but since there's no schema in JSON it basically just saves you some `", "` segments - around 25% of the original JSON size. A better choice is to use something like protobuf. Since you're specifying a schema upfront that you know consumers must have, all your keys can be represented by numbers. This saves you well over 50% of the original JSON size.

[Skipping some discussion about how precisely protobuf packs bits]

#### Schema evolution
Adding new fields to a protobuf schema is as easy as just giving them a new tag number. If old code parses it with the old schema, it just skips all new fields. New code parsing old data will fall back to the default or null value for any new field. Changing the data type of a field is a bit tricker because the parser has to cast (to the new type for new code, and to the old type for old code).

#### Avro
Avro is a binary encoding format that explicitly supports multiple compatible schemas. When you encode data with Avro, you specify (somewhere) the schema with which you wrote it. When you decode it, you pass the decoder the old schema and your new schema, and the decoder comes up with a translation between the two.

How does this differ from protobuf? Protobuf encodes tag numbers (i.e. some schema information) directly into the data. Avro doesn't contain any tag information at all! It just encodes a sequence of values, and relies on the reader possessing a copy of the writing schema to determine what the tags are and which tags map to which values. This makes Avro noticeably smaller than protobuf.

How does Avro tell the reader what the writer's schema is? For a large encoded file (like a Hadoop data file), the schema can just be put at the start of the file. For lots of individual records, you can write a version number that maps to a schema version. For communication over the network, the communicating process can negotiate schemas as part of the protocol (like how SSL certs are negotiated).

Besides space, one reason to prefer Avro is that you can more easily generate a dynamic schema from a dataset. With protobuf, you have to manually set field tags, being careful not to reuse them.

#### Dataflow through databases
Databases need to support schema changes for obvious reasons. One potential gotcha with choosing an encoding is that old code needs to preserve unfamiliar fields that are created by new code - otherwise an older replica will wipe all the fields it doesn't know about that were created by new replicas.

Since data outlives code, databases can often end up with lots of different schemas over a very long time. You can migrate all your data if you have to but you generally don't. Encodings that allow easy schema evolution (like Avro) can present a unified schema to the reader, even though the data itself is a menagerie of historical encodings.

When you're dumping data from the db into a data warehouse, you may as well reencode it to a consistent encoding. An Avro object container file (with the schema at the top) is a good fit.

#### Dataflow through services
Just as databases write data to a store and read it out, services communicating over HTTP write data to each other and read responses. As services change, their interfaces/schemas will change too. Old versions of a service should still be able to communicate to newer versions.

The two most popular protocols for communicating web services are REST and SOAP. The key idea of REST is to use as many HTTP features as possible: URLs for identifying resources, HTTP cache control/auth/content type negotiation. By contrast, SOAP is an attempt to build a protocol entirely on top of HTTP, avoiding the use of built-in HTTP features. 

##### RPC
One other protocol (or family of protocols) is RPC, or "remote procedure calls". The idea here is to make a callout to another service look the same as calling a function in your programming language. The book identifies a bunch of problems with this:

* Unlike a local function call, a network request can fail or be lost [lots of functions are like this though - e.g. writing to a local file]
* Local function calls return a result, crash, or timeout. RPCs can return with no result, due to a timeout, and you don't know what happened [aren't function timeouts the same?]
* Retrying a failed RPC can lead to duplicate actions if the request actually got through, forcing RPCs to be idempotent
* RPCs can take varying amounts of time to execute, especially in the worst case
* Local function calls can take data structures, but RPCs must encode everything
* RPCs must translate data types from one language to another

In short, RPCs and local function calls are fundamentally different [I disagree] and making them look similar is misleading.

[I'm a bit confused here. Isn't an API client class (e.g. in a Rails app) essentially building your own RPC on top of REST/HTTP? So you can call `account.billing_details` without the caller knowing about where the billing service lives? Isn't a db write effectively a RPC?]

Current ways to do RPC mitigate some of these problems by using async-friendly data types like promises and by providing service discovery.

Whatever encoding your RPC implementation uses for transport, it had better support some way of changing the schema!

#### Message-passing
REST and RPC both expect a response immediately. Another option is to use a message broker/message queue, which holds onto a message and handles delivery/service discovery/redelivery. Usually a reply is not expected (although it can be implemented via a separate channel). Examples: RabbitMQ, Kafka.

In general a process sends a message to a queue (or _topic_), and the broker ensures that all messages on that queue are delivered to some set of subscribers. There can be many producers and subscribers on one topic.

One advantage of a message queue is the ease of swapping in and out producers/consumers. Encoding is important here - the more flexible your schema can be, the easier it is to swap services out.

[Skipping section on distributed actor frameworks. Seems like a particular instance of RPC/REST, except between threads rather than machines on a network.]

