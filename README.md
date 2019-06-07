# Designing Data-Intensive Applications Notes

My notes on reading "Designing Data-Intensive Applications" by Martin Kleppman
https://www.amazon.com.au/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321

What follows is a very brief summary of each chapter. My more detailed chapter notes are in the relevant folders.

## Part 1: Foundations of Data Systems

### Reliable, scalable and maintainable applications
A "soft" chapter that defines some terms and clarifies the point of the book: to help programmers who work on systems that process or store data, which is almost every programmer and certainly every web programmer.

### Data models and query languages
Discusses some different ways of storing data - relational, like MySQL, or document, like Mongo - at a pretty high-level. Goes through what the query languages look like for these different models and some pros and cons. As with everything else in this book, it depends how you expect your data to be queried.

### Storage and retrieval
A lower-level discussion of how to store data efficiently. Mostly about the tradeoffs between slow, durable disk access and fast, ephemeral memory access. Compares two major options - LSM-trees and B-trees - in depth. Discusses how the choice of storage engine changes for data warehouses and analytics-focused data stores.

### Encoding and evolution
Goes into more detail about what "writing data to disk" means. Covers a bunch of ways to encode data to disk, from JSON to binary formats like Avro, and what tradeoffs each of these approaches make.

## Part 2: Distributed Data

Why distribute data? So that you can handle more data that fits on a single node, or keep working if a node dies, or serve data from nodes closer to where your users live. Making individual nodes bigger is easier but only addresses the first concern. This chapter deals with cases where you're scaling horizontally: many small nodes.

### Replication
Discusses the purpose and pitfalls of replicating data. Compares three main approaches to replication: single leader, multi leader, and leaderless. Describes three consistency requirements databases might fulfil - read-after-write, monotonic reads, and consistent-prefix reads - and how to get them. Anything other than single leader replication turns out to be very difficult to get right.

