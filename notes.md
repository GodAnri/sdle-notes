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
- Nodes establish `k` local contacts using some distance metric among vertices (e.g. in a ring) and a few long-range contacts uniformly at random, resulting in low diameter and high clustering;
- Flooding in Watts and Strogatz graph, we can achieve a `O(log N)` path; nevertheless, going to the next nearest point to our target can make us keep jumping and only achieve `O(sqrt(N))` paths, because these paths lack locality;
- Kleinberg solved this issue by choosing a probability function that made it more likely for links to be local;
- Long range contacts can be tuned to become more clustered in the vicinity;
- The goal is to have uniformity across all distance scales, a property found in DHT designs like Chord, and locally find `O(log`<sup>`2`</sup>`N)` routes;

-------------------

## System Design for Large Scale
### Motivation
- Fast adoption of a new service may kill it;
- Since many users have always-on broadband connections, it is tempting to use resources latent at the network edge (P2P);
- The more users that adopt a new service, the more power there is to run it (at least in theory - law of diminishing returns: in a production system with fixed and variable inputs, beyond some point, each additional unit of variable input yields less and less aditional output);

### Gnutella
**Early design:**
- Fully distributed solution to P2P file sharing;
- Partially randomised overlay network, each node connects to a number of nodes that varies across nodes and allocated bandwidths;
- Bootstrapping is done by HTTP hosted host caches (the system came with some known HTTP addresses pointing to servers with a list of IP and ports to machines working on the Gnutella network);
- Due to high churn, local host caches can quickly become outdated (these addresses either had to be sent with the source code or an website where users can update their list had to exist);
- Routing based on flooding (`PING`) and reverse path routing (`PONG`);
- Querying based on flooding (`QUERY`) and reverse path routing (`QUERY RESPONSE`) as well, the answer set grows with time until the diameter/maximum hop is reached;
- `GET` and `PUSH` requests are used to initiate file transfer directly between peers (`PUSH` is used to circumvent single firewalls that would block a `GET` in a given direction);
- This early design was not scalable (`PING`/`PONG` traffic was dominant).

**Improved:**
- Some nodes had higher maximum connectivity and bandwidth, most were always-on servers;
- Nodes preferred connection to these "super peers" with more uptime;
- Super peers work as in the early Gnutella, while shielding traffic;
- A digest (bloom filter) is sent from peers to super peers, which only contact target peers with a high likelihood of having the queried content;

**Bloom filter:**
- 256 bit code containing information about the presence of some word(s) in a text;
- Using a hash function multiple times with a different addition to the initial string (for example, SHA1(s + "0"), SHA1(s + "1"), etc.), the word(s) is(are) hashed and, for each of the variants, the bit corresponding to the 8-bit result is set to 1.
- A part of the query can be hashed and, if all bits corresponding to the hashes of that word are set to 1, the content is probably in the node;
- Longer queries lead to a higher likelihood of false positive.

### Distributed Hash Tables
Even in a super peer architecture, search in Gnutella is essentially flooding. DHTs are able to bound the number of hops traversed to lookup a given target, by providing a way of mapping keys to network nodes. In order to preserve some structure in the routing supporting the DHT, joins and leaves should be accounted for, so these protocols can require more maintenance than unstructured approaches.

**Chord:**
- Nodes and keys are assigned probabilistic unique ids from `0` to `2`<sup>`m`</sup>`-1`;
- Both node ids (e.g. IPs or e-mail addresses) and keys are hashed by SHA1 and `m` bits are taken, being assigned a `mod 2`<sup>`m`</sup>;
- The node whose ID is closest after a keyID will store that key;
- Each node keeps the address and id of clockwise `m` other nodes, and `r` vicinity nodes in both directions;
- The `i`<sup>`th`</sup> entry in node `n` indicates the first node that succeeds `n` by at least `2`<sup>`i-1`</sup>;
- Routing takes `O(log n)` steps, nodes keep `O(log n)` knowledge on other nodes.

**Kademlia:**
- Nodes and keys share a 160 bits id space, and keys are stored in "close by" nodes.
- Distance is measured with the XOR metric, which respects the triangle property (length of sides);
- Unlike Chord, routing is symmetric and alternative next hops can be taken for low latency or parallel routing;
- Routing tables consist of a list for each bit of the node ID, and a node in position `i` must have bits `0` to `i-1` identical to the owner and a different `i`<sup>`th`</sup> bit (so it is easy to find nodes for the first positions, but hard to find nodes for the last ones);
- To account for failing nodes and alternative paths, in each position up to `k` (about 20) nodes are stored;
- Candidate node uptimes are considered when competing for `k` limited positions.

-------------------

## Physical and Logical Time
### Time properties
- Time needs memory (since it is countable change);
- Time is local (rate of change slows with acceleration);
- Time synchronisation is harder at distance;
- Time is a bad proxy for causality.

### Clock drift
- Clock drift is the drift between measured time and a reference time;
- A second is defined in terms of transitions of Cesium-133;
- UTC (Coordinated Universal Time) inserts/removes seconds to sync with orbital time.

### Synchronisation
- **External synchronisation:** precision with respect to an external authoritative reference - guarantee that `|S(t) - C`<sub>`i`</sub>`(t)| < D`, for a band `D > 0` and a UTC source `S`;
- **Internal synchronisation:** precision among two nodes - guarantee that `|C`<sub>`j`</sub>`(t) - C`<sub>`i`</sub>`(t)| < D`, for a band `D > 0`;
- An externally synchronised system at band `D` is necessarily also at `2D` internal synchronisation;
- Some uses (e.g. `make`) require monotonicity (`t' > t`$\implies$`C(t') > C(t)`);
- Advanced clocks can be corrected by reducing the time rate until aimed synchrony is reached;

### Synchronisation in synchronous systems
- Knowing the transit time `trans` and receiving origin time `t`, one could set to `t + trans`;
- However, `trans` can vary between `tmin` and `tmax`.
- Using `t + tmin` or `t + tmax`, the uncertainty is `u = tmax - tmin`.
- Using `t + (tmin + tmax)/2`, the uncertainty becomes `u/2`.

### Synchronisation in asynchronous systems
- Transit time varies between `tmin` and +$\infty$, so there is no midpoint;
- **Cristian's Algorithm:**
    - Send a request `m`<sub>`r`</sub> that triggers response `m`<sub>`t`</sub> carrying time `t`;
    - Measure the RTT of request and reply as `t`<sub>`r`</sub>;
    - Set clock to `t + t`<sub>`r`</sub>`/2` assuming a balanced RTT;
    - Precision can be increased by repeating the protocol until a low `t`<sub>`r`</sub> occurs.
- **Berkeley Algorithm:**
    - A coordinator measures RTT to the other nodes and sets target time to the average of times;
    - New times for nodes to set are propagated as deltas relative to their local times, in order to be resilient to propagation delays.

### Causality: Happens-before
Causality is a partial order relation, and is only potential influence.
- Collect memories as sets of unique events;
- Set inclusion explains causality (`{a`<sub>`1`</sub>`,b`<sub>`1`</sub>`} `$\subset$` {a`<sub>`1`</sub>`,a`<sub>`2`</sub>`,b`<sub>`1`</sub>`}`);
- A simpler way of checking causality is checking the latest event (`e`<sub>`n `</sub>$\in$` C`<sub>`x`</sub>$\implies$`{e`<sub>`1`</sub>`,e`<sub>`2`</sub>`,...,e`<sub>`n`</sub>`} `$\subset$` C`<sub>`x`</sub>);

### Vector clocks
- `{a`<sub>`1`</sub>`,a`<sub>`2`</sub>`,b`<sub>`1`</sub>`,b`<sub>`2`</sub>`,b`<sub>`3`</sub>`,c`<sub>`2`</sub>`,c`<sub>`2`</sub>`,c`<sub>`3`</sub>`}` $\Leftrightarrow$ `{a` $\to$ `2, b` $\to$ `3, c` $\to$ `3}` $\Leftrightarrow$ `[2,3,3]`;
- Check of causality: `[a, b, c]` $\to$ `[a', b', c']` if `a` $\le$ `a'` and `b` $\le$ `b'` and `c` $\le$ `c'`;
- Concurrency, order and difference can be represented graphically;
- Message reception retrieves point-wise maximum and registers one new event on the result;
- Comparing vectors is linear on vector size, but can be improved by tracking the last event.

### Version vectors
- Not all events are registered, only the important ones, that cause versioning;
- Message reception does not register a new event;

### Scaling causality
- Data Center has `get/put` interface;
- Two proxies receive the same `get` and send a different `put` to the server;
- Three options:
    - Conditional writes: only the first `put` is accepted;
    - Overwrite first value: use clock (assuming synchronisation) to determine the last writer (Last-Writer-Wins, e.g. Cassandra);
    - Multi-Value: `[1,0]` $\parallel$ `[0,1]`, one entry per client (linear-size version client).
- **Scaling at the edge (Dotted Version Vectors):**
    - `get`s get all values, with compact causal context (`[0,0]s`<sub>`1`</sub> `U` `[0,0]s`<sub>`2`</sub> `= [2,0]`);
    - `put`s can go to alternative servers (server T can get a `put` with context `[2,0]`);
    - Allows to deal with clusters of servers (each with an entry on the version vector), and interactions through proxies without conditional writes or LWW;
- **Dynamic causality (Interval Tree Clocks)**
    - Tracking causality requires exclusive access to identities, and to avoid preconfiguring them, id space can be split and joined;
    - A seed node controls the initial id space;
    - Registering events can use any portion above the controlled id;
    - Ids can be split from any available entity, and these resulting entities can register new events and become concurrent;
    - Any two entities can merge together, and eventually all entities can collect the whole id space and simplify the encoding of events;

-------------------

## High Availability under Eventual Consistency
In geo-replication, latencies vary widely ($\lambda$` < 50ms` - local region DC; `100ms < `$\Lambda$` < 300ms` - intercontinental):
- Consensus/Paxos: [$\Lambda$,$2\Lambda$] (no divergence);
- Primary-Backup: [$\lambda$,$\Lambda$] (asynchronous/lazy);
- Multi-Master: $\lambda$ (divergence).

### EC and CAP
- **Eventually consistent:** ideally, when an update is made, all observers should see that update; building reliable distributed systems demands trade-offs between consistency and availability;
- **CAP theorem:** of three properties - data **c**onsistency, system **a**vailability and tolerance to network **p**artition - only two can be achieved at any given time; focus on **AP** (availability + partition tolerance);
- **Eventual consistency:** after an update, if no new updates are made to the object, eventually all reads will return the same value (e.g. DNS); this can be reformulated to avoid quiescence (for convergent, permanently changing systems) by adapting a session guarantee.
- **Session guarantees:**
    - Read your writes - read operations reflect previous writes;
    - Monotonic reads - successive reads reflect a non-decreasing set of writes;
    - Writes follow reads - writes are propagated after reads on which they depend (writes made during the session are ordered after any writes whose effects were seen by previous reads in the session);
    - Monotonic writes - writes are propagated after writes that logically precede them (writes are only incorporated into a server's database copy after all previous session writes).

### Sequential to concurrent execution
- Consensus and linearisability provides illusion of a single replica, which preserves (slow) sequential behaviour ($\lt$); 
- EC Multi-Master (or active-active) can expose concurrency ($\prec$);

### Conflict-Free Replicated Data Types (CRDTs)
- **Road to accomodate transition from sequential to concurrent:**
    - Permutation equivalence: if operations can commute sequentially, then they should preserve the same result under concurrency;
    - Preserving sequential semantics;
    - Concurrent executions can have richer outcomes;
- Convergence after concurrent updates; favor AP under CAP;
- Examples: counters, sets, mv-registers, maps, graphs;
- Operation-based CRDTs (operation effects must commute):
    - Example: PNCounter (`inc(dec(c)) = dec(inc(c))`); G-Set (`add`<sub>`a`</sub>`(add`<sub>`b`</sub>`(s)) = add`<sub>`b`</sub>`(add`<sub>`a`</sub>`(s))`).
- State-based CRDTs (rooted on join semi-lattices);
    - $\le$ reflects monotonic state evolution (increase of information).
- Eventual consistency non-stop:
    - `upds(a)` $\subseteq$ `upds(b)` $\implies$ `a` $\le$ `b` (weaker version of `upds(a) = upds(b)` $\implies$ `a = b`).
- Design of CRDTs:
    - Partially ordered logs (pologs) of operations implement any CRDT;
    - Any query at replica `i` can be expressed from local polog `O`<sub>`i`</sub>; example: counter at `i` is `|{inc` $\in$ `O`<sub>`i`</sub>`}| - |{dec` $\in$ `O`<sub>`i`</sub>`}|`.
- Implementing counters (e.g. CRDT PNCounters):
    - Track number of incs and decs done at each replica (`{A(inc,dec), B(inc,dec), C(inc,dec)}`);
    - At any point, counter value is `sum(inc) - sum(dec)`;
    - Joining does point-wise maximums among entries;
- Registers:
    - Registers are an ordered set of write operations (can be sequential under distribution);
    - Register value is the last written value;
    - LWW using timestamps is a simple approach to evolve state without strong coordination (needs synchronised wall-clocks);
    - Concurrency semantics show all concurrent values (e.g., values `k` and `y` are written concurrently -> `{k, y}` is the value when merging - multi-value registers; application-level merge needed to get final value);
    - Concurrency can be precisely tracked with version vectors;
    - Problem: concurrent `add(x)`, `rmv(x)`;
    - Add-wins: `x` is in the set if $\nexists$ `rmv(x)` $\prec$ `add(x)` (e.g. Redis CRDTs); using tombstone lists avoids concurrency problems.

-------------------

## Quorums
### Quorum Consensus Protocols
- Clients communicate directly to the servers/replicas;
- Each operation (e.g. read/write) requires a quorum (set of replicas);
- If the result of one operation depends on another one, their quorums must overlap (have common replicas);
- A simple way to define replicas is to consider all replicas as peers (quorums are determined by their size - similar to weighted voting with all weights = 1);

### Implementation
- Each object has a version number;
- **Read:** polls a read quorum, to find current version, then reads the value from an up-to-date replica;
- Output of a read operation depends on previous write operations, therefore, read and write quorums must overlap: `N`<sub>`R`</sub>`+N`<sub>`W`</sub> $\ge$ `N`;
- **Write:** polls a write quorum, to find current version, then writes the new value and new version to a write quorum;
- Since writes depend on previous writes (via version), and therefore write quorums must overlap: `N`<sub>`W`</sub>`+N`<sub>`W`</sub> $\ge$ `N`;

### Faults and consistency
- Partitions can cause different values for the same version and, when the partition is healed, clients reading can get a different value from their own write - does not ensure read-your-writes; also happens with concurrent writes;
- Consistency can be ensured with transactions, using two-phase commit or a variant (the client won't get the vote from the inaccessible replica and value won't be updated); at least a write quorum must accept the write; also solves the problem with concurrent writes using locks;
- Can lead to deadlocks, if transactions use locks, or blocking, if they use 2PC;
- Choosing `N`<sub>`R`</sub> and `N`<sub>`W`</sub> appropriately, we can trade off performance and availability of the different operations;
- Assigning each replica its own number of votes provides extra flexibility;
- Quorum consensus tolerates both replica and communication failures

### Dynamo
- Replicated key-value storage system with associative memory abstraction: `put(key,value)`, `get(key)`;
- Uses version vector instead of version number;
- Enhances high-availability, sacrificing strong consistency under certain failure scenarios;
- Each key is associated with a set of servers (preference list), the first `N` being the main replicas and the remaining being backup, used only in case of failures;
- Each operation has a coordinator from the `N` main replicas;
- `put()` requires a quorum of `W` replicas and `get()` requires a quorum of `R` replicas, such that `R + W `$\gt$` N`;
- `put(key, value, context)`: the coordinator determines the new version from `context`, a set of version vectors, and sends the `(key, value)` pair and the version vector to the `N` first servers in `key`'s preference list (deemed successful if at least `W-1` replicas respond);
- `get(key)`: the coordinator requests all versions of the `(key, value)`, from the remaining first `N` servers in the preference list and returns **all** the `(key, value)` pairs whose version vector is maximal after receiving `R-1` responses (if multiple, the application executing `get()` must reconcile versions and write-back the reconciled pair using `put()`);
- In case of failure, the coordinator will try to get a sloppy quorum by enlisting the backup replicas in the preference list.

-------------------

## Quorum Consensus and Replicated Abstract Data Types
### Initial and final quorums
- When executing an operation, a client:
    - Reads from an **initial quorum**;
    - Writes to a **final quorum**;
- Either can be empty;
- A quorum is any set of replicas that includes both an initial and a final quorum;
- Assuming all replicas are equal, a quorum may be represented by a pair `(m,n)`, `m` being the size of its initial quorum and `n` of its final one;
- Intersection constraints are defined between the final quorum of one operation and the initial quorum of another;

### Example: Gifford's read/write quorums
- Read/write operations are subject to two constraints:
    1. Each final quorum for write must intersect each initial quorum for read;
    2. Each final quorum for write must intersect each initial quorum for write;
- Quorum intersection graph:
    <br>┌───┐<br>│   v<br>│  write ────────> read<br>└───┘
- Minimal quorum choices (5 replicas):
    - Read: `| (1,0) | (2,0) | (3,0) |`
    - Write: `| (1,5) | (2,4) | (3,3) |`
- Gifford's based replicated queue:
    - A queue has two basic operations, enqueue and dequeue;
    - Enqueue: adds an item to the queue;
    - Dequeue:
        - Read an initial read quorum, to determine current version of the queue;
        - Read the state from an updated replica;
        - If the queue is not empty, remove the head, write the new queue state to a final write quorum and return the item removed;
        - If the queue is empty, raise exception.
    - Minimal quorum choices:
        - Enqueue: `| (1,5) | (2,4) | (3,3) |`
        - Dequeue: `| (1,5) | (2,4) | (3,3) |`
        - Abnormal:`| (1,0) | (2,0) | (3,0) |`
        - Only the last column makes sense in terms of availability;

### Herlihy's replicated ADT's
- Use timestamps instead of version numbers (reduces messages and quorum intersection constraints);
- Replicas keep logs instead of versions;
- Read: similar to version-based, except replica is up-to-date by timestamp instead of version;
- Write: no need to read from initial quorum, timestamp generation guarantees total order for whole state changes;
- Quorum intersection graph: write ────────> read
- Minimal quorum choices:
    - Read: `| (1,0) | (2,0) | (3,0) | (4,0) | (5,0) |`
    - Write: `| (0,5) | (0,4) | (0,3) | (0,2) | (0,1) |`
    - For reads more common than writes, left is better;
- Replicated event logs vs state:
    - Event: state change represented as a pair `[operation(args), outcome(results)]`, e.g. `[read(), ok(x)]` and `[write(x), ok()]`.
    - Event log: sequence of log entries;
    - Log entry: timestamped event (e.g. `t`<sub>`0`</sub> `: [enq(x); ok()]`);
    - Rather than replicate state, replicate event logs;

### Example: Herlihy's replicated queue
- Enqueue/dequeue operations are subject to two constraints:
    1. Every initial dequeue quorum must intersect every final enqueue quorum;
    2. Every initial dequeue quorum must intersect every final dequeue quorum;
- Dequeue:
    - Read logs from an initial dequeue quorum and creates a view (log obtained by merging entries of a set of logs, in timestamp order, and discarding duplicates);
    - Reconstruct the queue state and find item to return;
    - If the queue is not empty, record `deq` event, appending new entry to view, and send modified view to a final `deq` quorum (replicas merge this view with their local logs);
    - Return dequeued item or exception;
- Quorum intersection graph:
    <br>           ┌───┐<br>enqueue ────> dequeue  │<br>  │         │ ^   │<br>  │         │ └───┘<br>  │         v<br>  └───────> abnormal
- Minimal quorum choices (5 replicas):
    - Enqueue: `| (0,1) | (0,2) | (0,3) |`
    - Dequeue: `| (5,1) | (4,2) | (3,3) |`
    - Abnormal:`| (5,0) | (4,0) | (3,0) |`
    - Herlihy's approach allows `enq` to be more available at the expense of `deq`'s availability;
- Problem: logs and messages grow indefinitely;
- Solution: garbage collect logs:
    - If an item has been dequeued, all items with earlier timestamps have been dequeued;
    - Not enough to remove entries, otherwise some may be readded upon merging;
    - Is enough to keep the horizon timestamp (timestamp of the most recently dequeued item) and a log with only `enq` entries, whose timestamps are later than the horizon timestamp;

### Issues with replicated ADT's
- Timestamps:
    - Are supposed to be generated by clients and consistent with linearisability;
    - Herlihy's relies on transactions and hierarchical timestamps, so the problem is reduced to transaction ordering and, if using locks, serial order is usually determined at commit time;
    - If not using transactions, how do clients generate a timestamp if the initial quorum is empty?;
- Logs:
    - Must be garbage collected to bound the size of messages;
    - May be effective for Herlihy's queues but not for other replicated ADT's;

-------------------

## Byzantine Quorums
### Byzantine process
- Byzantine Failures: when a process arbitrarily deviates from its specifications;
- Byzantine Generals Problem (BGP):
    - Agreement: All non-faulty processes deliver the same message;
    - Validity: If the broadcaster is non-faulty, all non-faulty deliver its message;
    - There is no solution to the BGP in a system with 3 processes, if one of them may be byzantine, if messages are not signed (faulty processes must be less than a third of all processes).
- With cryptographic signatures, the node receiving the different messages can identify whether the byzantine process is the broadcaster (in which case the receiver sends the message as signed by the broadcaster) or the receiver (which can't sign the modified message properly) - the protocol must prevent replay attacks.

### Quorum consensus with byzantine replicas
- Considering a quorum for read/write operations, in which some servers may exhibit byzantine failures (report random version numbers);
- Each replica stores a replica of the variable and a timestamp, which is assigned by clients when the client writes the replica, from a set of timestamps that does not overlap the sets of all other clients (e.g. adding the client id to an integer);
- Liveness doesn't make sense due to asynchrony;
- Safety: any read that is not concurrent with writes returns the last value written in some serialisation of the preceding writes;
- Begin and end of events determine a partial order (linearisability of read/write);
- **Write** of value `v` by client c:
    - c obtains a set A with timestamps from all servers of some quorum Q;
    - c creates a timestamp higher than any of A's and any of its own previous ones;
    - c sends the update `<v,t>` to servers until it has received an Ack from every server in quorum Q';
    - Server updates its local `<v',t'>` if `t` $\gt$ `t'` and returns an Ack to the client in both cases;
    - Begins when c initiates the operation, ends when all correct servers in some quorum have processed the update `<v,t>`.
- **Read** of variable `x` by client c:
    - c obtains a set A with value/timestamp pairs from all servers of some quorum Q;
    - c applies a deterministic function `Result(A)` to obtain the result of the read operation;
    - Begins when c initiates the operation, ends when `Result(A)` function returns.
- Obtaining the reply from every server in the quorum is the only way to ensure that it gets the reply from all non-faulty servers;
- Quorums are defined such that there is always one quorum of non-faulty servers (otherwise, a byzantine server might not respond and the client couldn't progress);

### Byzantine masking quorums
- `Q`<sub>`1`</sub> is the latest write quorum;
- `B` is the set of byzantine servers;
- `Q`<sub>`2`</sub> is the read quorum;
- `(Q`<sub>`1`</sub>$\cap$`Q`<sub>`2`</sub>`)\B` are the up-to-date servers;
- `Q`<sub>`2`</sub>`\(Q`<sub>`1`</sub>$\cup$`B)` are the servers with stale values;
- `Q`<sub>`2`</sub>$\cap$`B` are the servers with arbitrary values;
- Properties:
    - M-consistency: it is possible to read the most recent value (not all servers in `Q`<sub>`1`</sub>$\cap$`Q`<sub>`2`</sub> are byzantine);
    - M-availability: there is always a quorum that does not include byzantine nodes.

### Size-based byzantine masking quorums
- M-consistency ensures that the client always obtains an up-to-date value, but it must identify which is it: being `f` be the bound on faulty servers, we need `f + 1` up-to-date non-faulty servers to outnumber the faulty servers, thus every pair of quorums must intersect in at least `2f + 1` servers, therefore, `2q - n` $\ge$ `2f + 1`;
- M-availability ensures that `n - f` $\ge$ `q`;
- Combining both expressions, we have that `2(n - f) - n` $\ge$ `2f + 1`, thus `n` $\ge$ `4f + 1` and `q = 3f + 1`.

### Size-based non-byzantine read/write quorums
1. `w` $\ge$ `f + 1` - ensures that survives failures;
2. `w + r` $\gt$ `n` - ensures that reads see most recent write;
3. `n - f` $\ge$ `r` - ensures read availability;
4. `n - f` $\ge$ `w` - ensures write availability;
- `2n - 2f` $\ge$ `r + w` from 3 and 4;
- `2n - 2f` $\gt$ `n` $\Leftrightarrow$ `n` $\gt$ `2f` from 2;
- Let `n = 2f + 1`, then `w = f + 1` and `r = f + 1`;
- Increasing `n` worsens performance (requires larger quorums) but increases fault tolerance (as `f` can increase).
- Read: c queries servers until it gets a reply from a set Q of `3f + 1` servers; let A = `{(v`<sub>`u`</sub>`,t`<sub>`u`</sub>`): u` $\in$ `Q}` be the set of value/timestamp pairs received from at least `f + 1` servers, the result of read is the value returned by `Result(A)` (the value of the pair with the highest timestamp in A);
- It is possible for a read to fail because of concurrent writes;

### Dissemination quorum systems
- Application in repositories of self-verifying information (e.g. repositories of public keys and blockchains);
- Instead of `n` $\ge$ `4f + 1`, needs `n` $\ge$ `3f + 1`;
- A read that is concurrent with one or more writes returns either the value written by the last preceding write or any of the values being written in a concurrent write.

-------------------

## Practical Byzantine Fault-Tolerance
### Impossibility of consensus with a faulty process (FLP's impossibility result)
- Safety: the decided value depends on the input values and no two correct processes decide differently;
- Liveness: every execution decides a value;
- In an asynchronous system in which at least one process may crash, there is no deterministic consensus that is both live and safe, even if the network is reliable (no message loss);
- In an asynchronous system we cannot distinguish a slow process from a crashed one;

### Model
- Network may delay, duplicate, deliver out of order or fail to deliver messages;
- Nodes are byzantine but independent;
- Processes use cryptographic techniques to prevent spoofing and replays, as well as to detect corrupted messages;
- Digestion function: `D(m)`;
- Message `m` signed by `i`;
- SMR can be implemented:
    - Each replica: maintains service state and executes deterministic operations;
    - Essentially behave like a non-replicated system (satisfies linearisability);
    - Tolerates `f` faulty replicas with `n = 3f + 1`;

### Protocol Overview (SMR)
- View is a numbered system configuration (replicas move through a succession of views, each of them having a leader `p = v mod n`);
- View changes occur upon suspicion of the current leader;
- Leader receives message from client, atomically broadcasts the request to all replicas (ensure total order), replicas execute the request and send reply;
- Client waits for replies with the same result (`t` and `r`) from `f + 1` replicas;
- If the client does not receive replies on time, it broadcasts the request to all replicas, which resend the reply or relay it to the leader; if the leader doesn't multicast the request, it will eventually be suspected of being faulty.

### Atomic Broadcast
- PBFT uses quorums to implement atomic multicast;
- Replicas collect quorum certificates (set with one message for each element in quorum); weak certificates are sets with at least `f + 1` messages from different replicas (e.g. reply certificate, set that a client needs to receive to return the result);
- Three-phase commit protocol:
    - Pre-prepare: leader assigns an increasing sequence number to the message `m` and multicasts the `<<pre-prepare, v, n, d>, m>` message to the replicas; upon receiving that message, a replica accepts it if it is in view `v`, the signatures in `m` and `pre-prepare` are valid, `d` is the digest of `m`, the sequence number `n` is between a low and a high water marks, `h` and `H`, and it hasn't accepted a `pre-prepare` message in view `v` with sequence number `n` with a different digest `d`;
    - Prepare: after accepting a `pre-prepare` message, every replica `i` (including the leader) send a `<prepare, v, n, d, i>` to all other replicas, which is accepted by any other replica if `v` is the replica's current view, its signature is correct and the sequence number is between `h` and `H`;
        - Prepared certificate: each replica collects `2f prepare` messages from different replicas; after collecting the prepared certificate, a replica has prepared the request;
        - Total order within a view: it is guaranteed, because `2f + 1` replicas need to accept pre-prepare to obtain a prepare-certificate, and the quorums of two prepare-certificates have at least `f + 1` common replicas, thus at least one non-faulty replica would have accepted two pre-prepare messages in the same view and with the same sequence number but different digests (impossible);
        - Ensuring order across view changes: replicas may collect prepared certificates in different views with the same sequence number (if a replica receives a prepared certificate, the leader is faulty and there is a view change, but the new leader did not receive that prepared certificate);
    - Commit: after collecting a prepared certificate, replica `i` multicasts `<commit, v, n, d, i>` to all replicas, which is accepted if the view, the signature and the sequence number are correct; a replica may accept a `commit` message before moving to the commit phase;
        - Commit certificate: each replica collects `2f commit` messages from different replicas; after collecting the prepared and commited certificates, a replica has committed the request;
        - Ensures that if a replica committed a request, that request is prepared by at least `f + 1` non-faulty replicas; with the view change protocol, this guarantees that non-faulty replicas agree on the sequence number of a committed request, even if they may commit in different views, and that any request committed by a non-faulty replica will be committed by all non-faulty replicas.
- Request and delivery execution: each replica executes the operation if it has committed the request and it has executed all requests with a lower sequence number; replicas send a reply to the client after executing the requested operation and discard requests whose timestamp is lower than the last reply they sent to the client (guarantees exactly-once);
- Garbage collection and checkpoints:
    - Problem: a replica cannot discard the messages with a sequence number as soon as it executes that request, because some replicas may have missed some messages; for replica repair/replacement or partition fix, we need state synchronisation (if the log is pruned, state transfer protocol is required);
    - Checkpoint: every `K` requests, a replica checkpoints its state and generates a proof of correctness of that checkpoint (that makes the checkpoint stable), and can discard from the log earlier checkpoints and messages;
    - A replica maintains the last stable checkpoint, one or more unstable ones and the current state;
    - Stable certificate: set of `f + 1 checkpoint` signed messages for sequence number `n` with the same digest (proves that at least one non-faulty replica has generated a checkpoint with sequence number `n` and digest `d`);
    - Water marks are set to:
        - `h`: `n` of the last stable checkpoint;
        - `H`: `h + L`, where `L` is a small multiple (e.g. 2) of the checkpoint period `K`.

### View Change Protocol
- Purpose: ensure liveness upon failure of the leader, while ensuring safety;
- Leader failure is suspected with a timer, which prevents replicas from waiting indefinitely;
- Upon timeout, replicas start changing to view `v + 1`, stop accepting messages other than `checkpoint`, `view-change` and `new-view`, and multicast a `<viewchange, v + 1, n, C, P, i>`, where:
    - `n`: sequence number of the last stable checkpoint known to `i`;
    - `C`: checkpoint's stable certificate;
    - `P`: set with prepared certificate for each request prepared with `n' > n`.
- Guarantees that a request with sequence number `n` that doesn't have the prepared certificate is retransmitted in the new view with the same sequence number.
- To avoid changing a view too early, view change timeouts are increasingly high;
- If a replica receives a set of `f + 1` valid `view-change` for views greater than its current one, it sends a `view-change` for the smallest view in the set, even if its timer hasn't expired.

### Byzantine quorums vs PBFT
- Compared with SMR, BQ appear to require fewer messages, but also has stronger assumptions, especially regarding the client;
- Furthermore, these quorum protocols support only read/write operations;
- In order to build more complex operations, the number of messages, as well as race conditions, would increase.

-------------------

## Blockchain
### Bitcoin
- Direct online payments without a trusted third party;
- Maintains a record of all transactions (money transfers) ever performed in a distributed fashion (blockchain);
- P2P network supporting broadcast;
- Set of accounts, each of which has public and private keys (owner-generated, no need for a PKI).
- Blockchain:
    - Blocks are a set of events (e.g. transactions) with maximum size of 1MB, containing a header with metadata;
    - The first block in the chain is the genesis block;
    - Blocks are appended to the blockchain head (most recently added block).
- Network:
    - Blockchain is replicated in each node of the network;
    - Peers have random connections to others;
    - Each node attempts to connect to 8 others, but the node degree can be much larger if it accepts incoming connections.
- Consensus:
    - Consensus is needed to agree on the blocks and their order;
    - Conventional byzantine algorithms (either byzantine quorums or PBFT);
    - It is difficult to know how many nodes there are;
    - It is fairly easy to create multiple identities (faulty - known as the Sybil attack).

### Bitcoin proof-of-work
- Cryptographic puzzle that takes a random large time (find a nonce to include in the block header such that the header's SHA-256 is smaller than a target);
    - Target can be tuned to adjust the difficulty (number of hashes needed is `2`<sup>`256`</sup>`/target`);
    - Adjusted every 2016 blocks (expected = 14 days) so that the expected time required is 10 minutes.
- Upon solving the PoW, a node broadcasts the new block, which is validated by other nodes (verify PoW and check transactions), and then added to the blockchain head and forwarded;
- The node stops working on the PoW for the block it was and starts working on the next block, which will follow the one just added;
- When a node receives a block, its chain may be missing some of its ancestors, which it must fetch and validate (the protocol synchronises efficiently).

### Block broadcasting with anti-entropy
- Upon validation, a node sends an `inv`(entory) message with a set of hashes of blocks it has;
- If the receiver doesn't have some of those blocks, it sends a `getdata` message with a list of the hashes of blocks it wants;
- The node sends each block in the list in its own `block` message;
- Each newly created block is inserted into the network by a miner using an unsolicited `block` message to one or more peers.
- Block propagation delay:
    - Block validation may add a significant delay (due to the need for disk access) and it is repeated at every hop;
    - Block propagation delay has a long tail distribution;
    - Improvements:
        - Reduction of diameter;
        - Faster validation (faster HW).

### Bitcoin forks
- Occur when two or more nodes add a different block at the head at the same time;
- Resolution is based on the expected amount of work (switch to a larger blockchain);
- There is no guarantee that a block will persist, but the likelihood of it being removed decreases with each added block (confirmation);
- At some point, Nakamoto added "code-based checkpointing" (the hash of a block that cannot be replaced - or any of those that precede it - is hardcoded in the software);
- Eventual consistency with high probability, assuming that the hash-power of an adversary is limited;
- Accidental forks depend mainly on the expected time to generate the PoW and the block propagation delay;
- Selfish mining strategies: after finding a PoW, a node can withhold its block until a competing block is sent, in order to replace it.

### Bitcoin scalability and energy consumption
- Broadcasting;
- Computationally intensive PoW;
- Small blocks (1MB maximum);
- Storage of the whole blockchain kept by all nodes (restrictions above limit the growth rate).
- Transaction bound:
    - Theoretical limit of 8 transactions/second;
    - Bitcoin parameter tuning can't make up for the difference in over 3 orders of magnitude to other systems:
        - Block size: increasing by an order of magnitude may cause growth rate of 500 GB/year;
        - PoW difficulty: increase the block rate to 1/minute makes forking much more frequent.
- Extremely low energy-efficiency: to be relevant and secure, it requires a huge hash-power;

### Proof-of-stake
- Ethereum wants to replace PoW with PoS;
- Run a "lottery" to decide which user adds the next block;
- Number of "tickets" = product of the amount of coins by the time that amount is held (integral of the amount of coins over time).

### Permissioned blockchains
- Implementation of smart contracts (code that may be executed upon some event added to the blockchain);
    - Main problem is ensuring consensus on the contents of each block and their order.
- PBFT can be used to maintain a replicated log (i.e. a blockchain) but its complexity is `O(n`<sup>`2`</sup>`)`, while bitcoin's PoW scales (kind of) to thousands of nodes;
- Taxonomy regarding authorisation to maintain, grow and access a blockchain:
    - Permissionless/public: anyone can read and grow a blockchain (e.g. Bitcoin);
    - Consortium: maintained by members of a consortium (each member may run a few nodes, responsible for persistence and growth of the blockchain; may use different read policies, from open-access to consortium-only);
    - Permissioned/private: single organisation controls which blocks are added.