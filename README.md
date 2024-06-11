
## Chapter 2 - Data Models and Query Languages

Historically, data models started out as one big tree (the hierarchial model), but that wasn't good at representing many-to-many relationships, so the relational model was invented to solve that problem. More recently, developers found that some applications don't fit well into the relational model either. New nonrelational "NoSQL" datastores have diverged into two main directions:

1. *Document databases* targets use cases where data comes in self-contained documents and relationships between documents are rare.

2. *Graph databases* go in the opposite direction, targeting use cases where anything is potentially related to everything.

Document and graph databases typically don't enforce a schema for the data they store, which can make it easier to adapt applications to changing requirements. However, your application most likely still assumes that data has a certain structure; it's just a question of whether the schema is explicit (enforced on write) or implicit (assumed on read).

<br>

## Chapter 3 - Storage and Retrieval

Storage engines fall into two broad categories: those optimized for transaction processing (OLTP), and those optimized for analytics (OLAP).

- OLTP systems are usually user-facing, meaning they may see a huge volume of requests. In order to handle the load, applications usually only touch a small number of records per query. The application requests records using some kind of key, and the storage engine uses an index to find the data for the requested key. Disk seek time is often the bottleneck here.

- Data warehouses and similar analytic systems are less well known, because they are primarily used by business analysts, not by end users. They handle a much lower volume of requests than OLTP systems, but each query is typically very demanding, requiring many millions of records to be scanned in a short time. Disk bandwidth (not seek time) is often the bottleneck here, and column oriented storage is an increasingly popular solution for this kind of workload.

<br>

OLTP storage engines follow two main schools of thought:

- The log structured school, which only allows appending to files and deleting obsolete files, but never updates a file that has been written.
- The update-in-place school, which treats the disk as a set of fixed-size pages that can be overwritten. B-trees are the biggest example of this philosophy, being used in all major relational databases and many nonrelational ones.

The key idea behind log-structured storage engines is to turn seemingly random-access writes into sequential writes on disk, which enables higher write throughput due to the performance characteristics of hard drives and SSDs.

<br>

When queries in analytic workloads require sequentially scanning across a large number of rows, indexes become much less relevant. Instead, it becomes important to encode data very compactly to minimize the amount of data that the query needs to read from disk.

<br>


## Chapter 4 - Encoding and Evolution

Encoding
- Ways of turning data structures into bytes on the network or bytes on disk

<br>

Rolling upgrades

Where a new version of a service is gradually deployed to a few nodes at a time, rather than deploying to all nodes simultaneously. Rolling updates allow new versions of a service to be released without downtime (encouraging frequent small releases over rare big releases) and make deployments less risky (allowing faulty releases to be detected and rolled back without affecting a large number of users).

<br>

Evolvability
- The ease of making changes to an application

<br>

There are several different encoding formats with different compatibility properties:

- Programming language-specific encodings are restricted to a single programming language, and often fail to support forwards and backwards compatibility.
- Textual formats like JSON, XML and CSV are widespread, and their compatibility depends on how you use them. They have optional schema languages which are sometimes helpful and sometimes a hinderance. These formats are somewhat vague about datatypes, so you have to be careful with things like numbers and binary strings.
- Binary schema-driven formats allow compact, efficient encoding with clearly defined forward and backward compatibility semantics. They can be useful for documentation and code generation in statically typed languages. However, these formats have the downside that data needs to be decoded before it is human readable.

<br>

There are several modes of dataflow, illustrating different scenarios in which data encodings are important:

- Databases, where the process writing to the database encodes the data and the process reading from the database decodes it
- RPC and REST APIs, where the client encodes the request, the serve decodes the request and encodes a response, and finally the client decodes the response
- Asynchronous message passing (using message brokers or actors), where nodes communicate by sending each othe rmessages that are encoded by the sender and decoded by the recipient

<br>

## Chapter 5 - Replication

Replication serves several purposes:

*High availability*
- Keeping the system running, even when one or more node goes down

*Disconnected operation*
- Allowing an application to continue working when there is a network interruption

*Latency*
- Placing data geographically close to users, so that users can interact with it faster

*Scalability*
- Being able to handle a higher volume of reads than a single machine could handle, by performing reads on replicas

<br>

There are three main approaches to replication:

*Single-leader replication*

Clients send all writes to a single node (leader), which sends a stream of data change events to the other replicas (followers). Reads can be performed on any replica, but reads from followers may be stale.

<br>

*Multi-leader replication*

Clients send writes to one of many leader nodes, any of which can accept writes. Each leader sends streams of data change events to other leaders and to any follower nodes.

<br>

*Leaderless replication*

Clients send each write to several nodes, and read from several nodes in parallel in order to detect and correct nodes with stale data.

<br>

Single-leader replication is fairly easy to understand and there is no conflict resolution to wrry about. Multi-leader and leaderless replication can be more robust in the presence of network interruptions, faulty nodes, and latency spikes - at the cost of being harder to reason about and providing only very weak consistency guarantees.

Replication can be synchronous or asynchronous, which has a profound effect on system behavior when there is a fault. Asynchronous replication can be fast when the system is running smoothly, but if a leader fails and you promote an asynchronously updated follower to be the new leader, recently committed data may be lost.

<br>

There are a few consistency models to help decide how an application should behave under replication lag:

*Read-after-write-consistency*
- Users should always see data that they submitted themselves

*Monotonic reads*
- After users have seen the data at one point in time, they shouldn't later see the data from some earlier point in time

*Consistent prefix reads*
- Users should see the data in a state that makes causal sense: for example, seeing a question and its reply in the correct order


<br>

## Chapter 6 - Partitioning

Partitioning

Splitting up a large dataset into smaller subsets. Partitioning is necessary when you have so much data that storing and processing it on a single machine is no longer feasible.

The goal of partitioning is to spread the data and query load evenly across multiple machines, avoiding hotspots (nodes with disproportionally high load).

<br>

*Key range partitioning*

Keys are sorted, and partitions own all keys from some minimum up to some maximum. Sorting has the advantage that efficient range queries are possible, but there is a risk of hot spots if the application frequently accesses keys that are close together in sorted order.

Partitions are usually rebalanced dynamically by splitting the range into two subranges when the partition grows too large.

<br>

*Hash Partitioning*

A hash function is applied to each key, and a partition owns a range of hashes. This method destroys the ordering of keys, making range queries inefficient, but may distribute load more evenly.

When partitioning by hash, it is common to create a fixed number of partitions in advance, assign several partitions to each node, and move entire partitions from one node to another when nodes are added or removed.

<br>

Secondary indexes can also be partitioned:

*Document-partitioned indexes (local indexes)*

The secondary indexes are stored in the same partition as the primary key and value. This means only a single partition needs to be updated on write, but a read of the secondary index requires a scatter/gather across all partitions.

<br>

*Term-partitioned indexes (global indexes)*

Secondary indexes are partitioned separately, using the indexed values. An entry in the secondary index may include records from all partitions of the primary key. When a document is written, several partitions of the secondary index need to be updated, but a read can be served from a single partition.

<br>

## Chapter 7 - Transactions

Transaction

An abstraction layer that reduces certain concurrency problems and faults down to transaction aborts. The application just needs to retry in the event of an abort.

<br>

### Race conditions:

*Dirty Reads*

One client reads another client's writes before they have been committed. The read committed isolation level and stronger levels prevent dirty reads.

<br>

*Dirty Writes*

One client overwrites another client's uncommitted writes. Almost all transaction implementations prevent dirty writes.

<br>

*Read Skew*

A client sees different parts of the database at different points in time. Some cases of read skew are known as nonrepeatable reads. This issue is most commonly prevented with snapshot isolation, which allows a transaction to read from a consistent snapshot corresponding to one particular point in time. It is usually implemented with multi version concurrency control (MVCC).

<br>

*Lost Updates*

Two clients concurrently perform a read-modify-write cycle. One overwrites the other's write without incorporating its changes, leading to lost data. Some implementations of snapshot isolation prevent this automatically, while others require a manual lock (SELECT FOR UPDATE).

<br>

*Write Skew*

A transaction reads some data, makes a decision based on the value it saw, and writes the decision to the database. However, by the time the write is made, the premise of the decision is no longer true. Only serializable isolation prevents this anomaly.

<br>

*Phantom reads*

A transaction reads objects that match some search condition. Another client makes a write that affects the result of that search. Snapshot isolation prevents straightforward phantom reads, but phantoms in the context of write skew requires special treatment, such as index-range locks.

<br>

### Implementations of serializable transactions:

*Literally executing transactions in serial order*

If you can make each stransaction very fast to execute, and the transaction throughput is low enough to process on a single CPU core, this is a simple and effective option.

<br>

*Two-phase locking*

Two levels of locks, where multiple clients can hold read locks, but no one else can have a lock for a client to obtain to a write lock. Many applications avoid using it because of its performance characteristics.

<br>

*Serializable Snapshot Isolation (SSI)*

Snapshot isolation where transactions are aborted at time of commit if the execution was not serializable. It uses an optimistic approach, allowing transactions to proceed without blocking.


