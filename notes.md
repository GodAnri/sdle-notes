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
+ **Connected component:** maximal connected subgraph;
+ **Strongly connected graph:** a directed graph that, for each pair of nodes `(u,v)`, has a path from `u` to `v` and a path from `v` to `u`;
- **Tree:** connected graph with no cycles;
- **Star:** central vertice and many leaf nodes connected to it (includes trees);
- **Planar graph:** vertices and edges can be drawn in a plane and no two edges intersect (includes stars and trees);
+ **Degree of a node:** number of adjacent vertices; in directed graphs, there's in-degree and out-degree;
+ **Distance between nodes:** length of the shortest path connecting those nodes;
+ **Eccentricity of a node:** maximum distance from a node to all nodes in the network;
+ **Diameter of a graph:** maximum eccentricity of all nodes in a graph (maximum distance between any two nodes);
+ **Radius of a graph:** minimum eccentricity of all nodes in a graph;
+ **Center:** vertex/vertices with eccentricity equal to the radius;
    + Center of a tree: 1 or 2;
    + Center of a path: 1 or 2;
    + Center of a ring: number of nodes in the ring;
+ **Periphery:** vertex/vertices with eccentricity equal to the diameter;
    + Periphery of a ring: number of nodes in the ring.

### Network
In a network context, cyclic graphs allow multi-path routing, which can be more robust but data handling can become more complex. Thus, distributed algorithms often construct trees to avoid cycles, while others try to work under multi-path.

### Complex topologies
- **Random geometric:** vertices are dropped randomly uniformly into a unit square and edges are added to connect any two points within a given euclidean distance;
- **Random Erdos-Renyi `G(n,p)`:** `n` nodes are connected randomly; each edge is included with independent probability `p`;
- **Watts-Strogatz:** Manhattan grid + Erdos-Renyi;
- **Barabasi-Albert:** based on preferential attachment - the more connected a node is, the more likely it is to receive new links; degree distribution follows a power law - few nodes with high degree, many nodes with low degree;

### Spanning trees
- **Directed spanning tree:** a directed spanning tree with root node `i` is breadth-first if each node at distance `d` from `i` appears at depth `d` in the tree; every strongly connected graph has a breadth-first directed spanning tree;
- **SyncBFS algorithm:** initial state - `parent: nil, marked: False` (except for root, where `marked: True`); unmarked processes receiving a `search` message from `x` do `parent = x, marked = True` and, in the next round, `search` messages are sent from these;
    - **Complexity:**
        - **Time:** at most `diam` rounds (depends on `i` eccentricity);
        - **Message:** at most `|E|` messages are sent (across all edges);
    - **Child pointers:** if parents need to know their offspring, processes must reply with either `parent` or `nonparent`; this is only easy when the graph is undirected, but is achievable in general strongly connected graphs;
    - **Termination:** all processes respond with `parent` or `nonparent`; parent terminates when all children terminate; responses are collected from leaves to root;
    - **Applications:**
        - **Aggregation:** input values in each process can be aggregated towards a sync node; each value only contributes once; many functions can be used (sum, max, avg, voting, ...);
        - **Leader election:** largest `UID` wins; all processes become the root of their own tree and aggregate a `Max(UID)`; each decides by comparing its own `UID` with `Max(UID)`;
        - **Broadcast:** message payload can be attached to SyncBFS construction (`m|E|` message load) or broadcasted once tree is formed (`m|V|` message load);
        - **Computing diameter:** each process constructs a SyncBFS and determines `maxdist` (longest tree path); afterwards all processes use their trees to aggregate `max(maxdist)` from all roots/nodes; time: `O(diam)`; messages: `O(diam*|E|)`.
- **AsynchSpanningTree algorithm:** reliable FIFO with send/receive with cause() function linking a send to a receive; upon receiving a message, the node sets its `parent` and, instead of sending the message to every other neighbour apart from its parent, a `sendto` condition is set to `True` for each of them, making it *possible* to send the message; AsynchSpanningTree is not a direct asynchronous translation of SyncBFS, because it does not necessarily produce a breadth-first spanning tree (faster long paths will win over slower direct paths);
    - **Invariants:**
        - In any reachable state, the edges defined by all `parent` variables form a spanning tree of a subgraph of G, containing the root `i0`; moreover, if there is any message in any channel `C`<sub>`i,j`</sub>, then `i` is in this spanning tree;
        - In any reachable state, if `i` is a part of the tree (is root or has parent), and if `j` is a neighbour of `i` except the root, then either `j` is a part of the tree or `C`<sub>`i,j`</sub> contains a `search` message or `sendto(j) = true`;
    - **Theorem:** the AsynchSpanningTree algorithm constructs a spanning tree in an undirected graph;
    - **Complexity:** although time is unbounded, it is practical to assume an upper bound on time to execute an effect (`l`) and to deliver a message in a channel (`d`)
        - **Time:** `O(diam*(l+d))`;
        - **Messages:** `O(|E|)`;
        - Although a tree with height `h > diam` can occur, it only happens if it doesn't take more time than a tree with `h = diam`.
    - **Child pointers:** same as SyncBFS, even though the time complexity of the tree built from the `parent` variables can be at most `O(n(l+d))`, because a fast path is not always fast, it can be faster when calculating but slower afterwards;
    - **Applications:**
        - **Broadcast with Acks:** it is possible to build an AsynchBcastAck algorithm that collects Acks as the tree is constructed; upon incoming broadcast message, each node Acks if it already knows the broadcast, and Acks to parent once all neighbours Ack to it;
        - **Leader election with AsynchBcastAck:** AsynchBcastAck includes termination and if all nodes initiate it and report their UIDs, it can be used for leader election with unknown diameter and number of nodes.

### Epidemic Broadcast Trees
- Gossip Broadcast: highly scalable and resilient, but excessive message overhead.
- Tree-Based Broadcast: small message complexity, but fragile in presence of failures.

+ Eager push: nodes immediately forward new messages;
+ Pull: nodes periodically query for new messages;
+ Lazy push: nodes push new message ids and accept pulls (there is a separation among payload and metadata);

Epidemic broadcast tries to combine the best of both broadcast models:
- Nodes sample a few random peers into an `eagerPush` set (neighbours);
- Connections should be stable, can use TCP and links are reciprocal (bidirectional);
- First message reception puts origin in `eagerPush`, while further duplicate receptions move source to `lazyPush`;
- Eager push of payload, lazy push of metadata;
- If the tree breaks and graph stays connected, nodes get metadata but not payloads; this is detected by timer expiration and the metadata source is moved from `lazyPush` to `eagerPush`;
- Potential redundancy (due to cycles) is cleared later by the standard algorithm.

### Watts and Strogatz
- "Six degrees of separation" - every two people need an average of 6 known people hops to be related;
- Random graphs are not a good model for people acquaintances, since people graphs have more clustering;
- Watts and Strogatz proposed a model that mixes short-range and long-range contacts;
- Nodes establish `k` local contacts using some distance metric among vertices (e.g. in a ring) and a few long-range contacts uniformly at random, resulting in low diameter and high clustering.