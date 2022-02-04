# SDLE notes
## MOM - Message-Oriented Middleware

(...)

### MOM vs UDP
- Both MOM and UDP allow unicast and multicast (by IP multicast group, in the case of UDP).
- MOM allows asynchrony between the senders/publishers and the receivers/subscribers, as well as anonimity (queues and topics use high-level names rather than addresses).


Asynchronous communication is appropriate for applications in which the sender and receiver are **loosely coupled** (e.g. microservices, workflow applications or message-based communication - email, SMS, instant messaging).

JMS queues: sender must block (or use a callback) to ensure that the messages are delivered.

-------------------

## Processing and Scaling
### Threads
Abstract the execution of a sequence of instructions (function), whereas processes abstract the execution of a program.

Operations like creation, termination or switching are much more efficient on threads than on processes.

May share most resources, thread-specific information is relatively small: stack, processor state and thread state (ready, running or waiting).

Can be implemented by OS (kernel-level) or by user-level code (e.g. libraries).

### Kernel-level Threads
- The OS keeps a **threads table** with information on every thread. A process' control block points to its own threads table.
- All thread management operations, such as thread creation, incur a system call.

### User-level Threads
- Kernel is unaware of their existence.
- Library must provide thread creation, destruction, synchronisation and yield a core to other threads.

### User-level vs Kernel-level
**\+** The OS doesn't have to support threads;

**\+** The kernel is not involved in creation/destruction or switching;

**\-** Page-fault by one thread prevents others from running (the single kernel-level thread is *WAITING*);

**\-** Cannot exploit parallelism in multicore architectures.

### Hybrid implementation
The library maps user-level threads to kernel-level ones (number of user-level threads may be much larger than kernel-level).

### Multi-threaded implementation
- Each thread processes a request.
- When one thread blocks in I/O, another one is scheduled to run in its place.
- One dispatcher that accepts a connection request, several workers, each of which processes all the requests sent in the scope of a single connection.

### Thread-pools
- Allow to bound number of threads.
- Avoid overhead of creating/destroying threads (if you use a fixed minimum number of threads).
- Can cause excessive thread-switch overhead (especially when using multiple thread pools), may want to bound the number of active threads using a counting semaphore.

### Synchronous I/O
**Blocking:** Thread blocks until completion (write()/send() system calls may return immediately after copying the data to kernel space and enqueueing the output request).

**Non-blocking:** The call returns immediately with whatever data is available.

In UNIX, all I/O to block devices is blocking, even with the O_NONBLOCK flag.

### Asynchronous I/O
Enqueue the I/O request and return immediately. The thread may execute while the I/O operation is being executed, then learns about the termination by polling or event notification.

-------------------

## Replication and Consistency
### Advantages
- Replication increases the performance of a system, as well as its fault-tolerance and availability.

### Consistency
Updating at different replicas may lead to inconsistent data (conflicts).

### Strong consistency
All replicas execute updates in the same order - same initial state leads to same result.

### Models
**Sequential consistency:**
- An execution is sequential consistent if all operations executed by any thread appear in the order in which they were executed by the corresponding thread.
- This is the model provided by multi-threaded system on a uniprocessor.
- Reads from one replica, writes to every replica, snapshots from one replica.
- It is not composable (the combined execution of two sequential consistent algorithms may not be sequential consistent).

**Linearisability:**
- An execution is linearisable if it is sequential consistent and, if `op1` occurs before `op2`, according to an omniscient observer, then `op1` appears before `op2`.
- Assuming operations with a `start time` and a `finish time`, measured on a global clock, `op1` occurs before `op2` if `op1`'s finish time is smaller than `op2`'s start time.
- If `op1` and `op2` overlap in time, their relative order can be any.
- Reads from one replica, writes to and waits for ACK from every replica before returning, snapshots from one replica.

**One-copy serialisability (transaction-based systems):**
- An execution of a *set of transactions* is one-copy serialisable if its outcome is similar to the execution of the same transactions in a single copy.
- DB systems nowadays provide weaker consistency models to achieve higher performance.
- Equivalent to sequential consistency but with all operations being transactions (isolation ensures that the outcome of concurrent transactions is equal to a sequential execution of the same transactions).

-------------------

## Scalable Distributed Topologies
### Graphs
- **Walk:** edges and vertices can be repeated;
- **Trail:** only vertices can be repeated;
- **Path:** no repeated edges or vertices;
+ **Complete graph:** each pair of vertices has an edge connecting them;
+ **Clique:** complete sub-graph;
+ **Connected graph:** there is a path between any two nodes;
- **Tree:** connected graph with no cycles;
- **Star:** central vertice and many leaf nodes connected to it (includes trees);
- **Planar graph:** vertices and edges can be drawn in a plane and no two edges intersect (includes stars and trees);
+ **Connected component:** maximal connected subgraph;
+ **Degree of a node:** number of adjacent vertices; in directed graphs, there's in-degree and out-degree;
+ **Distance between nodes:** length of the shortest path connecting those nodes;
+ **Eccentricity of a node:** maximum distance from a node to all nodes in the network;
+ **Diameter of a graph:** maximum eccentricity of all nodes in a graph (maximum distance between any two nodes);
+ **Radius of a graph:** minimum eccentricity of all nodes in a graph;
+ **Center:** vertex/vertices with eccentricity equal to the radius;
+ **Periphery:** vertex/vertices with eccentricity equal to the diameter.

Examples
- Center of a tree: 1 or 2.
- Center of a path: 1 or 2.
- Center of a ring: number of nodes in the ring.
- Periphery of a ring: number of nodes in the ring.

### Network
In a network context, cyclic graphs allow multi-path routing, which can be more robust but data handling can become more complex. Thus, distributed algorithms often construct trees to avoid cycles, while others try to work under multi-path.
