## Chapter 1 - Reliable, Scalable, and Maintainable Applications

*Functional Requirements* are what an application should do, such as allowing data to be stored, retrieved, searched, and processed in various ways.

*Nonfunctional Requirements* are general properties like security, reliability, compliance, scalability, compatibility, and maintainability.

*Reliability* means making systems work correctly, even when faults occur. Faults can be in hardware (typically random and uncorrelated), software (bugs are typically systematic and hard to deal with), and humans (who inevitably make mistakes from time to time). Fault tolerance techniques can hide certain types of faults from the end user.

*Scalability* means having strategies for keeping performance good, even when load increases. In order to discuss scalability, we first need ways of describing load and performance quantitatively. Load can be described with numbers called load parameters, such as requests per second to a web server, the ratio of reads to writes in a database, etc. Response time percentiles is a way of measuring performance. In a scalable system, you can add processing capacity in order to remain reliable under high load.

*Maintainability* is in essence about making life better for the engineering and operations teams who need to work with the system. Good abstractions can help reduce complexity and make the system easier to modify and adapt for new use cases. Good operability means having good visibility into the system's health, and having effective ways of managing it.

<br>

## Chapter 2 - Data Models and Query Languages

Historically, data models started out as one big tree (the hierarchical model), but that wasn't good at representing many-to-many relationships, so the relational model was invented to solve that problem. More recently, developers found that some applications don't fit well into the relational model either. New nonrelational "NoSQL" datastores have diverged into two main directions:

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
- Textual formats like JSON, XML and CSV are widespread, and their compatibility depends on how you use them. They have optional schema languages which are sometimes helpful and sometimes a hindrance. These formats are somewhat vague about datatypes, so you have to be careful with things like numbers and binary strings.
- Binary schema-driven formats allow compact, efficient encoding with clearly defined forward and backward compatibility semantics. They can be useful for documentation and code generation in statically typed languages. However, these formats have the downside that data needs to be decoded before it is human readable.

<br>

There are several modes of dataflow, illustrating different scenarios in which data encodings are important:

- Databases, where the process writing to the database encodes the data and the process reading from the database decodes it
- RPC and REST APIs, where the client encodes the request, the serve decodes the request and encodes a response, and finally the client decodes the response
- Asynchronous message passing (using message brokers or actors), where nodes communicate by sending each other messages that are encoded by the sender and decoded by the recipient

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

Single-leader replication is fairly easy to understand and there is no conflict resolution to worry about. Multi-leader and leaderless replication can be more robust in the presence of network interruptions, faulty nodes, and latency spikes - at the cost of being harder to reason about and providing only very weak consistency guarantees.

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

If you can make each transaction very fast to execute, and the transaction throughput is low enough to process on a single CPU core, this is a simple and effective option.

<br>

*Two-phase locking*

Two levels of locks, where multiple clients can hold read locks, but no one else can have a lock for a client to obtain a write lock. Many applications avoid using it because of its performance characteristics.

<br>

*Serializable Snapshot Isolation (SSI)*

Snapshot isolation where transactions are aborted at time of commit if the execution was not serializable. It uses an optimistic approach, allowing transactions to proceed without blocking.

<br>

## Chapter 8 - The Trouble with Distributed Systems

There are a wide range of problems that can occur in distributed systems:

- Whenever you try to send a packet over the internet, it may be lost or arbitrarily delayed. Similarly, the response may be lost or delayed, so if you don't get a reply, you have no idea whether the message got through or not.
- A Node's clock may be significantly out of sync with other nodes, it may suddenly jump forward or back in time, and relying on it is dangerous because you most likely don't have a good estimate of your clock's confidence interval.
- A process can stop for a substantial amount of time at any point in time in its execution (perhaps due to a stop-the-world garbage collector), be declared dead by other nodes, and then come back to life again without realizing it was paused.

The fact that *partial failures* can occur is a defining characteristic of distributed systems. We try to build fault tolerances into the software, so that the system as a whole can continue functioning even when some of its constituent parts are broken.

To tolerate faults, the first step is to *detect* them, but even that is hard. Most systems don't have an accurate mechanism for determining if a node has failed, so most distributed algorithms rely on timeouts to determine whether a remote node is still available. However, timeouts can't distinguish between network and node failures, and variable network delay sometimes causes a node to be falsely declared dead. Moreover, sometimes a node can be in a degraded state but not dead, which can be even more difficult to deal with than a cleanly failed node.

Once a fault is detected, making a system tolerate it is not easy either: there is no shared memory, global variable, no common knowledge or any shared state between the machines. Nodes can't even agree on what time it is, let alone anything more profound. The only way information can flow from one node to another is by sending a message over an unreliable network. Major decisions cannot be made safely by a single node, so we require protocols that enlist the help of other nodes to try to get a quorum to agree.

<br>

## Chapter 9 - Consistency and Consensus

*Linearizability*

A popular consistency model whose goal is to make replicated data appear as though there were only a single copy, and to make all operations act on it atomically. It is appealing because it makes the database behave like a variable in a single-threaded program, but it has the downside of being slow, especially in environments with large network delays.

<br>

*Causality*

Imposes an ordering on events in a system (what happened before what, based on cause and effect). Unlike linearizability, which puts all operations in a single, totally ordered timeline, causality provides a weaker consistency model: some things can be concurrent, so the version history is like a timeline with branching and merging. Causal consistency does not have the coordination overhead of linearizability and is much less sensitive to network problems.

<br>

Causal ordering (such as using Lamport timestamps) is still not enough for solving some problems, such as claiming unique usernames. For a node to accept a username registration, it needs to know that another node isn't concurrently in the process of registering the same name. This problem leads to *consensus*.

Achieving consensus means deciding in some way that all nodes agree on what was decided, and the decision is irrevocable. A wide range of problems are actually reducible to consensus and are equivalent to each other:

*Linearizable compare-and-set registers*
- The register needs to atomically *decide* whether to set its value, based on whether its current value equals the parameter given in the operation

*Atomic transaction commit*
- A database must *decide* whether to commit or abort a distributed transaction

*Total order broadcast*
- The messaging system must *decide* on the order in which to deliver messages

*Locks and leases*
- When several clients are racing to grab a lock or lease, the lock *decides* which one successfully acquired it

*Membership/coordination service*
- Given a failure detector (e.g. timeouts), the system must *decide* which nodes are alive, and which should be considered dead because their sessions timed out

*Uniqueness constraint*
- When several transactions concurrently try to create conflicting records with the same key, the constraint must *decide* which one to allow and which should fail with a constraint violation

All of these are straightforward if you only have a single node, or assign the decision-making capability to a single node. This happens in a single-leader database, and they are able to provide linearizable operations, uniqueness constraints, a totally ordered replication log, and more.

However, if that single leader fails, or if a network interruption makes the leader unreachable, such a system becomes unable to make any progress. There are three ways of handling that situation:

1. Wait for the leader to recover, and accept that the system will be blocked in the meantime. This approach does not fully solve consensus because it does not satisfy the termination property: if the leader does not recover, the system can be blocked forever.

2. Manually fail over by getting humans to choose a new leader node and reconfigure the system to use it. Many relational databases take this approach. The speed of failover is limited by the speed at which humans can act, which is generally slower than computers.

3. Use an algorithm to automatically choose a new leader. This approach requires a consensus algorithm, and it is advisable to use a proven algorithm that correctly handles adverse network conditions.

Tools like ZooKeeper provide an "outsourced" consensus, failure detection, and membership service that applications can use.

Nevertheless, not every system necessarily requires consensus: leaderless and multi-leader replication systems typically do not use global consensus. The conflicts that occur in these systems are a consequence of not having consensus across different leaders, but maybe that's okay.

<br>

## Chapter 10 - Batch Processing

The Unix philosophy includes design principles of inputs being immutable, outputs intended to become the input of other programs, and complex problems being solved by composing small tools that do one thing well. These principles are carried forward into MapReduce and more recent dataflow engines.

In the Unix world, the uniform interface that allows one program to be composed with another is files and pipes; in MapReduce, that interface is a distributed filesystem. Dataflow engines add their own pipe-like data transport mechanisms to avoid materializing intermediate state to the distributed filesystem, but the initial input and final output of a job are still usually files in HDFS.

<br>

There are two main problems that batch processing frameworks need to solve:

*Partitioning*
- In MapReduce, mappers are partitioned according to input file blocks. The output of mappers is repartitioned, sorted, and merged into a configurable number of reducer partitions. The purpose of this process is to bring all the related data (e.g., all records of with the same key) together in the same place.
- Post-MapReduce dataflow engines try to avoid sorting unless it is required, but they otherwise take a broadly similar approach to partitioning.

*Fault tolerance*
- MapReduce frequently writes to disk, which makes it easy to recover from an individual failed task without restarting the entire job but slows down execution in the failure-free case. Dataflow engines perform less materialization of intermediate state and keep more in memory, which means they need to recompute more data if a node fails. Deterministic operators reduce the amount of data that needs to be recomputed.

There are several join algorithms for MapReduce that are also internally used in MPP databases and dataflow engines:

*Sort-merge joins*
- Each of the inputs being joined goes through a mapper that extracts the join key. By partitioning, sorting, and merging, all the records with the same key end up going to the same call of the reducer. This function can then output the joined records.

*Broadcast hash joins*
- One of the two join inputs is small, so it is not partitioned and it can be entirely loaded into a hash table. Thus, you can start a mapper for each partition of the large join input, load the hash table for the small input into each mapper, and then scan over the large input one record at a time, querying the hash table for each record.

*Partitioned hash joins*
- If the two join inputs are partitioned in the same way (using the same key, same hash function, and same number of partitions), then the hash table approach can be used independently for each partition.

<br>

Distributed batch processing engines have a deliberately restricted programming model: callback functions are assumed to be stateless and to have no externally visible side effects besides their designated output. This restriction allows the framework to hide some of the hard distributed systems problems behind its abstraction: in the face of crashes and network issues, tasks can be retried safely, and the output from any failed tasks is discarded. If several tasks for a partition succeed, only one of them actually makes its output visible.

The distinguishing feature of a batch processing job is that it reads some input data and produces some output data, without modifying the input - in other words, the output is derived from the input. Crucially, the input data is *bounded*: it has a known, fixed size (for example, it consists of a set of log files at some point in time, or a snapshot of a database's contents). Because it is bounded, a job knows when it has finished reading the entire input, and so a job eventually completes when it is done.

<br>

## Chapter 11 - Stream Processing

*AMQP/JMS-style message broker*
- The broker assigns individual messages to consumers, and consumers acknowledge individual messages when they have been successfully processed. This approach is appropriate as an asynchronous form of RPC, for example in a task queue, where the exact order of message processing is not important and where there is no need to go back and read old messages again after they have been processed.

*Log-based message broker*
- The broker assigns all messages in a partition to the same consumer node, and always delivers messages in the same order. Parallelism is achieved through partitioning, and consumers track their progress by checkpointing the offset of the last message they have processed. The broker retains messages on disk, so it is possible to jump back and reread old messages if necessary.

The log-based approach has similarities to the replication logs found in databases and log-structured storage engines. This approach is especially appropriate for stream processing systems that consume input streams and generate derived state or derived output streams.

Streams can come from user activity events, sensors providing periodic readings, and data feeds like market data in finance. It can also be useful to think of writes to a database as a stream: the changelog - the history of all changes made to a database - can be captured implicitly through change data capture or explicitly through event sourcing. Log compaction allows the stream to retain a full copy of the contents of a database.

Representing databases as streams allows you to keep derived data systems like search indexes, caches, and analytics systems continually up to date by consuming the log of changes and applying them to the derived system. You can even build new views onto existing data by starting from scratch and consuming the log of changes from the beginning all the way to the present.

The facilities for maintaining state as streams and replaying messages are also the basis for the techniques that enable stream joins and fault tolerance in various stream processing frameworks.

There are three types of joins that may appear in stream processes:

*Stream-stream joins*
- Both input streams consist of activity events, and the join operator searches for related events that occur within some window of time. For example, it may match two actions taken by the same user within 30 minutes of each other.

*Stream-table joins*
- One input stream consists of activity events, while the other is a database changelog. The changelog keeps a local copy of the database up to date. For each activity event, the join operator queries the database and outputs an enriched activity event.

*Table-table joins*
- Both input streams are database changelogs. In this case, every change on one side is joined with the latest state of the other side. The result is a stream of changes to the materialized view of the join between the two tables.

As with batch processing, we need to discard the partial output of any failed tasks. But, since a stream is long-running and produces output continuously, we can't simply discard all output. Instead, a finer-grained recovery mechanism can be used, based on microbatching, checkpointing, transactions, or idempotent writes.

<br>