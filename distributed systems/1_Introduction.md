# Distributed Systems: Unit 1 Notes (Introduction)

---

## 1. Introduction to Distributed Systems

### 1.1 Definition
A **distributed system** is a collection of independent computers (nodes) that appears to its users as **a single coherent system**. The nodes communicate **only by passing messages** over a network.

> Leslie Lamport's joke: *"A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable."*

### 1.2 Key Characteristics

| Characteristic | Meaning |
|---|---|
| **No shared memory** | Nodes coordinate only through messages |
| **No global clock** | Each node has its own clock, so event ordering is hard |
| **Concurrency** | Many components run at the same time |
| **Independent failures** | One node or link can fail without the others failing |
| **Heterogeneity** | Different hardware, OS, languages, networks |
| **Autonomy** | Each node has its own local control |

### 1.3 Goals (Why build one?)
1. **Resource sharing**: printers, files, databases, compute
2. **Transparency**: hide the distribution from users
3. **Openness**: standard interfaces, so parts can be added or replaced
4. **Scalability**: handle growth in users, data, and geography
5. **Fault tolerance and availability**: keep working when parts fail
6. **Performance**: parallelism and locality

### 1.4 Advantages and Disadvantages

| Advantages | Disadvantages |
|---|---|
| Better price/performance | Complex design and debugging |
| Scalability (add nodes) | Network dependence (latency, partitions) |
| Reliability and fault tolerance | Security risks (more attack surface) |
| Resource sharing | No global state or clock |
| Geographic distribution | Software is harder to write |
| Incremental growth | Partial failure is hard to detect |

### 1.5 Centralized vs Parallel vs Distributed

| Feature | Centralized | Parallel | Distributed |
|---|---|---|---|
| Memory | Single | Shared | Private (per node) |
| Communication | N/A | Shared memory | Message passing |
| Goal | Simplicity | Speed | Sharing, availability, scale |
| Failure | Total | Usually total | Partial |
| Clock | One | Often common | Independent |

### 1.6 Middleware
Software layer between the OS and applications. It hides heterogeneity and gives a uniform programming model (RPC, messaging, naming, transactions).

```
+-----------+  +-----------+  +-----------+
|   App A   |  |   App B   |  |   App C   |
+-----------+--+-----------+--+-----------+
|             MIDDLEWARE                   |
+-------------------------------------------+
|  OS (Linux) |  OS (Windows) |  OS (macOS) |
+-------------+---------------+-------------+
|         Network (TCP/IP)                  |
+-------------------------------------------+
```

### 1.7 Real-World Examples
- The Internet and WWW
- Google Search, Gmail, YouTube
- Amazon, Flipkart (e-commerce backends)
- Cassandra, MongoDB, HDFS
- Blockchain networks
- UPI and banking systems

---

## 2. Distributed Computing System Models

### 2.1 Physical (Classic) Models
These are the traditional models for organizing machines in a distributed computing system.

#### a) Minicomputer Model
Several minicomputers, each with multiple users on terminals. Resources (printers, databases) are shared across machines through a network.

```
 [Terminals]   [Terminals]   [Terminals]
     |             |             |
 [Mini-1] ----- [Mini-2] ----- [Mini-3]
        \          |          /
         ====== Network ======
```
- **Pros:** simple extension of time-sharing
- **Cons:** weak load balancing, tied to terminals

#### b) Workstation Model
Each user has a personal workstation (own CPU, memory, disk). Idle workstations can be used by others.

```
 [WS1] [WS2] [WS3] [WS4]
   |     |     |     |
   ======= LAN ========
```
- **Pros:** good response time, user autonomy
- **Cons:** idle machines are wasted, and moving processes is complex (needs process migration)
- Example: Sprite OS

#### c) Workstation-Server Model
Diskless or low-end workstations plus **dedicated servers** (file server, print server, database server).

```
 [WS1] [WS2] [WS3]        [File Server]
   |     |     |          [DB Server]
   ======= LAN ========== [Print Server]
```
- **Pros:** common in practice, centralized management, scalable
- **Cons:** servers can be bottlenecks or single points of failure
- Uses **client-server** request/response

#### d) Processor-Pool Model
Users have only thin terminals. A **pool of processors** is allocated dynamically per task.

```
 [Terminal]   [Terminal]            +--------------+
     |            |                 | CPU CPU CPU  |
     ===== Network ===== [Run Server]| CPU CPU CPU  |
                                    +--------------+
```
- **Pros:** efficient processor use, flexible allocation, good for parallel jobs
- **Cons:** poor for interactive or graphics workloads
- Example: Amoeba OS

#### e) Hybrid Model
Combines workstation-server and processor pool: interactive work runs locally, heavy computation goes to the pool.

### 2.2 Architectural Models

| Model | Description | Example |
|---|---|---|
| **Client-Server** | Clients request, servers respond | Web (browser and server) |
| **Peer-to-Peer (P2P)** | All nodes are equal and act as both client and server | BitTorrent |
| **Multi-tier (3-tier)** | Presentation, application, data layers | Web app, app server, DB |
| **Layered** | Each layer uses the one below | OSI/TCP-IP |
| **Hybrid** | Mix of the above | Skype, modern CDNs |

```
 Client-Server            Peer-to-Peer
 C1   C2   C3             P1 --- P2
  \   |   /                |  \ /  |
   [Server]                P4 --- P3
```

### 2.3 Fundamental Models (System Models, Coulouris)
This is the most "interview-relevant" part. It abstracts a system's essential properties.

#### a) Interaction Model: Synchronous vs Asynchronous

| Property | Synchronous | Asynchronous |
|---|---|---|
| Message delay | Known upper bound | Unbounded |
| Process speed | Known bounds | No bounds |
| Clock drift | Known bound | Arbitrary |
| Failure detection | Possible with timeouts | Impossible to do reliably |
| Realism | Idealized | Closer to the Internet |

> Consensus is impossible in a purely asynchronous system with even one crash failure (**FLP impossibility**).

#### b) Failure Model

| Failure Type | Description |
|---|---|
| **Crash (fail-stop)** | Process halts and stays halted |
| **Omission** | Message or response is lost (send/receive omission) |
| **Timing** | Response outside the allowed time |
| **Arbitrary (Byzantine)** | Any behavior, including malicious or incorrect output |
| **Link failure** | Message lost, duplicated, reordered, or corrupted |
| **Network partition** | Network splits into groups that cannot communicate |

Severity order: **Crash < Omission < Timing < Byzantine**

#### c) Security Model
Threats: eavesdropping, masquerading, tampering, replay, denial of service. Defenses: encryption, authentication, digital signatures, access control.

---

## 3. Design Issues

### 3.1 Transparency
Hiding the complexity of distribution from users and programmers.

| Type | Hides |
|---|---|
| **Access** | Differences in data representation and how a resource is accessed |
| **Location** | Where a resource physically is |
| **Migration** | That a resource may move to another location |
| **Relocation** | That a resource moves while in use |
| **Replication** | That multiple copies exist |
| **Concurrency** | That many users share a resource simultaneously |
| **Failure** | Failure and recovery of a resource |
| **Persistence** | Whether a resource is in memory or on disk |
| **Performance** | Reconfiguration to balance load |
| **Scaling** | Growth of the system without changing its structure |

*Example:* Opening `https://amazon.in` hides which of the thousands of servers handled you (location and replication transparency).

> Full transparency is impossible and sometimes undesirable (you cannot hide network latency or the speed of light).

### 3.2 Reliability and Fault Tolerance
- **Reliability:** continuity of correct service
- **Availability:** fraction of time the system is usable
- **Techniques:** redundancy (hardware, software, time, information), replication, checkpointing and recovery, failure detection (heartbeats, timeouts)

$$\text{Availability} = \frac{\text{MTTF}}{\text{MTTF} + \text{MTTR}}$$

### 3.3 Flexibility
- **Monolithic kernel:** everything in the kernel, fast but rigid
- **Microkernel:** minimal kernel, services run in user space, which is modular and easier to extend (preferred in distributed OSes)

### 3.4 Performance
- Minimize message count and size
- Use caching, replication, batching
- Exploit locality
- Minimize latency and maximize throughput
- Use **fine-grained vs coarse-grained parallelism** wisely (less communication means better performance)

### 3.5 Scalability
Dimensions: **size** (users/resources), **geographic** (distance), **administrative** (multiple organizations).

| Technique | Idea |
|---|---|
| **Hiding latency** | Asynchronous communication, do other work while waiting |
| **Distribution** | Split a component and spread it out (e.g., DNS) |
| **Replication and caching** | Copies reduce load and latency (but need consistency) |

Scalability pitfalls: centralized services, centralized data, centralized algorithms.

### 3.6 Heterogeneity
Differences in networks, hardware, OS, programming languages, and data formats. Solved by **middleware**, standard protocols (TCP/IP, HTTP), and data serialization (JSON, Protocol Buffers, XML).

### 3.7 Openness
Published, well-defined interfaces (IDL), standardized protocols, and extensibility, so third-party components can integrate.

### 3.8 Security
Confidentiality, integrity, availability, authentication, authorization, and non-repudiation. Secure channels (TLS), firewalls, and access control lists.

### 3.9 Other Design Issues
- **Naming:** DNS, URLs
- **Resource management:** load balancing, scheduling, process migration
- **Synchronization:** clocks, mutual exclusion, deadlock
- **Consistency and replication:** keeping copies in sync
- **Concurrency control:** transactions, locking
- **Global state and ordering of events**

---

## 4. Communication Primitives

### 4.1 Basic Message Passing: `send` and `receive`
```
send(destination, message)
receive(source, buffer)
```

#### Classification of primitives

| Axis | Option A | Option B |
|---|---|---|
| **Blocking** | **Blocking send:** the sender waits until the message is buffered or delivered | **Non-blocking send:** returns immediately |
| **Synchronization** | **Synchronous:** sender waits for the receiver to receive | **Asynchronous:** sender continues after copying the message to a buffer |
| **Buffering** | **Buffered:** the system stores messages until received | **Unbuffered:** the receiver must be ready, otherwise the message is dropped or the sender waits |
| **Reliability** | **Reliable:** acknowledgment, retransmission | **Unreliable:** best effort, no guarantee |

**Blocking receive:** waits until a message arrives. **Non-blocking receive:** returns immediately (polling or interrupt).

```
Synchronous (blocking) send         Asynchronous (non-blocking) send
 Sender        Receiver              Sender        Receiver
   |--- msg --->|                      |--- msg --->|
   |  (waits)   | process              | continues  | process
   |<--- ack ---|                      |            |
   | continues                         |
```

### 4.2 Sockets (Practical Message Passing)
- **TCP (stream):** reliable, ordered, connection-oriented
- **UDP (datagram):** unreliable, unordered, connectionless, low overhead

| TCP | UDP |
|---|---|
| Connection setup (3-way handshake) | No connection |
| Reliable, ordered | Best effort |
| Flow and congestion control | None |
| HTTP, FTP, SSH | DNS, video streaming, gaming |

### 4.3 Remote Procedure Call (RPC)
Lets a program call a procedure on a **remote machine** as if it were local, hiding the message passing.

#### RPC Flow
```
 CLIENT MACHINE                         SERVER MACHINE
+---------------+                      +----------------+
| Client code   |                      | Server code    |
|   result =    |                      |  proc(x,y)     |
|   add(2,3)    |                      |                |
+------+--------+                      +-------^--------+
       | 1.call                               | 5.call
+------v--------+                      +-------+--------+
| Client Stub   |                      | Server Stub    |
| marshal params|                      | unmarshal      |
+------+--------+                      +-------^--------+
       | 2.                                   | 4.
+------v--------+     3. network       +-------+--------+
|  RPC Runtime  | -------------------> |  RPC Runtime   |
+---------------+ <------------------- +----------------+
        (reply travels back the same way: 6 -> 7 -> 8)
```

#### Steps
1. Client calls the **client stub** (a normal local call)
2. Stub **marshals** (packs) parameters into a message
3. Message is sent to the server over the network
4. Server's **stub unmarshals** the parameters
5. Stub calls the actual **server procedure**
6. Result is marshalled and sent back
7. Client stub unmarshals the result
8. Control returns to the client code

#### Key Terms
- **Stub/Proxy:** local placeholder that hides networking
- **Marshalling:** converting data to a transmittable format; **unmarshalling** is the reverse
- **IDL (Interface Definition Language):** defines the interface, and stubs are generated from it
- **Binding:** client finds the server (via a name or binder service)

#### RPC Call Semantics (Failure Handling), a very common question

| Semantics | Meaning | Mechanism | Example |
|---|---|---|---|
| **Maybe** | May execute once or not at all | No retry | Fire and forget |
| **At-least-once** | Executes one or more times | Retry on timeout | Suitable for **idempotent** ops (read) |
| **At-most-once** | Executes zero or one time | Retry plus duplicate filtering | Most RPC systems (gRPC, Java RMI) |
| **Exactly-once** | Executes exactly once | Very hard, since it needs failure-proof messaging | Approximated with idempotency keys |

#### RPC Problems
- **Parameter passing:** pointers and references don't work across address spaces, so copy-restore or deep copy is used
- **Heterogeneity:** data representation (endianness), solved by standard encoding
- **Failures:** client crash (orphans), server crash, lost request or reply
- **Higher latency** than a local call, and it can never be fully transparent

### 4.4 Remote Method Invocation (RMI)
The object-oriented version of RPC (Java RMI). It invokes methods on **remote objects** and can pass object references.

| RPC | RMI |
|---|---|
| Procedure-oriented | Object-oriented |
| Passes data only | Passes objects and references |
| Language-independent (usually) | Often language-specific (Java) |

### 4.5 Group Communication (Multicast)
One-to-many or many-to-many communication.

- **Unicast:** one to one
- **Broadcast:** one to all
- **Multicast:** one to a group

#### Message Ordering in Multicast

| Ordering | Guarantee |
|---|---|
| **FIFO** | Messages from the same sender arrive in the order sent |
| **Causal** | If message A caused message B, then A is delivered before B everywhere |
| **Total** | All receivers see messages in the **same** order |

Reliability: **reliable multicast** ensures all members receive the message or none do (atomic multicast).

### 4.6 Message-Oriented Middleware (MOM)
Uses **message queues**, so sender and receiver need not be online at the same time (**asynchronous, loosely coupled**). Examples: RabbitMQ, Kafka, AWS SQS.

```
 Producer --> [ Queue / Broker ] --> Consumer
```

### 4.7 Comparison Summary

| Feature | Message Passing | RPC | Message Queue |
|---|---|---|---|
| Abstraction | Low level (send/recv) | Procedure call | Queue (pub/sub) |
| Coupling | Tight | Tight in time | **Loose** |
| Sync | Either | Usually synchronous | Asynchronous |
| Use | Custom protocols | Microservices (gRPC) | Event-driven systems |

---

## 5. Likely Interview Questions

1. **What is a distributed system? Give examples.**
   A set of independent computers that appear as one system and communicate via messages, e.g., Google Search, banking, Cassandra.

2. **Difference between distributed system and parallel system?**
   Parallel: tightly coupled, shared memory, focused on speed. Distributed: loosely coupled, message passing, focused on sharing, availability, and scalability.

3. **What are the types of transparency?**
   Access, location, migration, relocation, replication, concurrency, failure, persistence, performance, scaling.

4. **What is the difference between synchronous and asynchronous distributed systems?**
   Synchronous has bounded delays and clock drift. Asynchronous has none, which makes failure detection impossible and consensus impossible in the worst case (FLP).

5. **Explain RPC and its working.**
   Client stub marshals parameters, sends them over the network, the server stub unmarshals and calls the procedure, and the result returns the same way.

6. **What are RPC failure semantics?**
   Maybe, at-least-once, at-most-once, exactly-once.

7. **What is idempotency and why does it matter?**
   An operation that gives the same result if executed multiple times (e.g., `set x = 5`, not `x = x + 1`). It makes at-least-once retries safe.

8. **Blocking vs non-blocking send?**
   Blocking waits for delivery or buffering. Non-blocking returns immediately and the program continues.

9. **TCP vs UDP?**
   TCP is reliable, ordered, and connection-based. UDP is fast, unreliable, and connectionless.

10. **What is middleware?**
    A layer that hides heterogeneity and provides services like RPC, naming, and messaging.

11. **Why is scalability hard? How do you achieve it?**
    Centralized bottlenecks. Use distribution, replication and caching, and latency hiding.

12. **What are the types of failures in a distributed system?**
    Crash, omission, timing, Byzantine (plus link failures and partitions).

13. **RPC vs RMI?**
    RPC calls procedures. RMI calls methods on remote objects and passes object references.

---

## 6. Quick Revision Sheet

- **Distributed system:** independent nodes, message passing only, no shared memory, no global clock, partial failure
- **Goals:** sharing, transparency, openness, scalability, fault tolerance
- **Models:** Minicomputer, Workstation, Workstation-Server, Processor-Pool, Hybrid
- **Architectures:** client-server, P2P, multi-tier
- **Fundamental models:** interaction (sync/async), failure (crash, omission, timing, Byzantine), security
- **Design issues:** transparency, reliability, flexibility, performance, scalability, heterogeneity, openness, security
- **Scalability techniques:** hide latency, distribute, replicate/cache
- **Primitives:** send/receive, blocking vs non-blocking, buffered vs unbuffered, reliable vs unreliable
- **RPC:** stub, marshalling, IDL, binding. Semantics: maybe < at-least-once < at-most-once < exactly-once
- **Group communication:** FIFO, causal, total ordering
- **MOM:** queues for loose coupling (Kafka, RabbitMQ)

