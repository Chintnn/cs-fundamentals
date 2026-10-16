# Distributed Systems: Unit 2 Notes (Synchronization)

---

## 1. Why Synchronization Is Hard

- There is **no shared memory** and **no global clock**, so nodes cannot simply compare timestamps.
- Each node's physical clock **drifts**, and message delays are variable.
- Two questions drive this unit:
  1. **What time is it?** (clock synchronization)
  2. **What happened before what?** (logical clocks, global state)
- Then come the coordination problems: **who is the leader?** (election) and **who may use the resource?** (mutual exclusion).

---

## 2. Clock Synchronization

### 2.1 Basic Terms

| Term | Meaning |
|---|---|
| **Clock skew** | Difference between two clocks at one instant |
| **Clock drift** | Rate at which a clock runs faster or slower than real time |
| **Drift rate (ρ)** | Maximum drift, for example 10⁻⁶ s/s (about 1 second per 11.6 days) |
| **UTC** | Coordinated Universal Time, the global reference |

If a clock's drift rate is at most ρ, and we want two clocks to differ by no more than δ, they must resynchronize at least every **δ / 2ρ** seconds.

### 2.2 Cristian's Algorithm (external, centralized)

A client asks a **time server** (which has an accurate source such as UTC) for the time.

```
 Client                         Time Server
   | T0 --- "what's the time?" -->  |
   |                                | (reads clock = Ts)
   | T1 <------ Ts -----------------|
   |
   RTT = T1 - T0
   New client time = Ts + RTT/2
```

**Simulation output:**
```
Cristian: RTT=0.100s, set clock = Ts + RTT/2 = 12.100
```
With Ts = 12.050 and RTT = 0.100, the client sets its clock to 12.050 + 0.050 = **12.100**.

- **Assumption:** request and reply delays are symmetric.
- **Accuracy:** ± (RTT/2 − min delay).
- **Problems:** single point of failure, and a faulty server gives wrong time to everyone.
- **Rule:** never set the clock backwards. Slow it down gradually instead, so time does not go back and break ordering.

### 2.3 Berkeley Algorithm (internal, no UTC needed)

A **master** polls all slaves, **averages** the times (ignoring outliers), then sends each machine the **adjustment** to make, not the new time. Sending adjustments avoids the message-delay error.

**Simulation output:**
```
Master: 3:00  adjust by +300s
S1:     3:25  adjust by -1200s
S2:     2:50  adjust by +900s
average = 3:05
```

| Cristian | Berkeley |
|---|---|
| External sync (to UTC) | Internal sync (agree among themselves) |
| Passive time server | Active master polls clients |
| Sends the time | Sends the adjustment |

### 2.4 Network Time Protocol (NTP)

Used on the Internet. It uses a **hierarchy of servers (strata)**, with Stratum 0 being atomic or GPS clocks, Stratum 1 servers connected to them, and so on.

With timestamps T1 (client sends), T2 (server receives), T3 (server replies), T4 (client receives):

$$\text{Offset } \theta = \frac{(T2 - T1) + (T3 - T4)}{2} \qquad \text{Delay } \delta = (T4 - T1) - (T3 - T2)$$

Accuracy is about milliseconds over the Internet and sub-millisecond on a LAN.

### 2.5 Other Methods
- **Berkeley, NTP, SNTP** (simple NTP)
- **Reference Broadcast Synchronization (RBS)** for wireless sensor networks
- **Precision Time Protocol (PTP)** for microsecond accuracy
- **Google TrueTime** (used in Spanner) returns a time **interval** [earliest, latest] using GPS and atomic clocks

---

## 3. Logical Clocks

**Key idea (Lamport):** for many problems we only need the **order of events**, not the real time.

### 3.1 Happened-Before Relation (→)

1. If `a` and `b` are events in the **same process** and `a` occurs before `b`, then **a → b**.
2. If `a` is **sending** a message and `b` is **receiving** it, then **a → b**.
3. **Transitivity:** a → b and b → c implies a → c.
4. If neither a → b nor b → a, the events are **concurrent** (a ‖ b).

### 3.2 Lamport Logical Clocks

**Rules (each process Pi has a counter Ci):**
1. Before every local event: `Ci = Ci + 1`
2. When sending, attach the timestamp: `ts = Ci` (after incrementing)
3. On receiving a message with timestamp `ts`: `Ci = max(Ci, ts) + 1`

**Simulation output:**
```
P1: local a                clock=1
P1: send m1->P2            clock=2
P2: local b                clock=1
P2: recv m1                clock=max(own,2)+1=3
P2: send m2->P3            clock=4
P3: local c                clock=1
P3: recv m2                clock=max(own,4)+1=5
```

```
 P1:  a(1)---send(2)
                    \
 P2:      b(1)       recv(3)---send(4)
                                     \
 P3:   c(1)                           recv(5)
```

**Property:** a → b implies C(a) < C(b).
**Limitation:** C(a) < C(b) does **not** imply a → b. Lamport clocks cannot detect concurrency.

**Total ordering:** break ties with process ID, using the pair (Ci, i), so every pair of events gets a unique order. This is used in mutual exclusion and replicated state machines.

### 3.2.1 Vector Clocks

Each process keeps a **vector** V of size N, where V[j] is "the number of events of Pj that I know about."

**Rules (process Pi):**
1. Local event or send: `V[i] = V[i] + 1`
2. Send: attach the whole vector
3. Receive vector `W`: `V[k] = max(V[k], W[k])` for all k, then `V[i] = V[i] + 1`

**Comparing vectors:**
- V ≤ W if every V[k] ≤ W[k]
- V < W if V ≤ W and V ≠ W, which means **V happened before W**
- Neither V ≤ W nor W ≤ V means **concurrent**

**Simulation output:**
```
A event   [1, 0, 0]
A send    [2, 0, 0]
B recv    [2, 1, 0]
C event   [0, 0, 1]
a1 vs b1: a -> b (happened-before)
b1 vs c1: CONCURRENT
```
The vectors [2,1,0] and [0,0,1] are incomparable, so the events are concurrent.

| Feature | Lamport Clock | Vector Clock |
|---|---|---|
| Size | 1 integer | N integers |
| a → b ⇒ C(a) < C(b) | Yes | Yes |
| C(a) < C(b) ⇒ a → b | **No** | **Yes** |
| Detects concurrency | No | **Yes** |
| Cost | Low | O(N) per message |
| Use | Ordering, mutual exclusion | Causal ordering, conflict detection (Dynamo, Riak) |

---

## 4. Global State

### 4.1 The Problem
There is no common memory, so we cannot take an instantaneous photograph of the whole system. Needed for **deadlock detection, garbage collection, checkpointing, termination detection, debugging**.

### 4.2 Key Concepts
- **Local state:** state of one process
- **Channel state:** messages sent but not yet received
- **Global state:** the set of all local states plus all channel states
- **Cut:** a line across the time diagram that splits each process's events into past and future
- **Consistent cut:** if a receive event is in the cut, its corresponding send event is also in the cut. No message travels from the future into the past.
- **Inconsistent cut:** contains a receive whose send is outside it, so the state is impossible.

```
 Consistent cut             Inconsistent cut
 P1: ---*---|--------       P1: ---*-----|------
         \  |                           \|
 P2: -----\-|-------       P2: ---------|\|----
           \|  (send inside)            | recv inside,
                                         send outside -> BAD
```

### 4.3 Chandy-Lamport Snapshot Algorithm

Records a **consistent global state**. Assumes **FIFO, reliable channels** and no failures.

**Steps:**
1. **Initiator** records its own state and sends a special **MARKER** on all outgoing channels.
2. When a process receives a MARKER on channel `c` for the **first time**:
   - Records its own local state
   - Marks channel `c` as **empty**
   - Sends MARKERs on all its outgoing channels
   - Starts recording messages arriving on all **other** incoming channels
3. When it receives a MARKER on a channel it has **already** been recording: **stop** recording that channel. The messages recorded are that channel's state.
4. The snapshot is complete when markers have arrived on all incoming channels at every process.

**Properties:** works while the application continues running, uses O(E) marker messages (E = number of channels), and gives a consistent (though maybe never-occurred) state.

**Example (bank accounts):** P1 has ₹100, P2 has ₹50, and a transfer of ₹10 is in transit. The snapshot records P1 = 90, P2 = 50, channel = {₹10 in transit}, so the total is **150**, with no money lost or double counted.

---

## 5. Coordination

Coordination covers protocols where processes must **agree** on something or take turns:
- **Who is the leader?** (election)
- **Who enters the critical section?** (mutual exclusion)
- **Do all processes commit or abort?** (distributed commit/consensus, covered in later units)

Most coordination protocols need **failure detection**, typically by heartbeats and timeouts. In an asynchronous system this can only be approximate.

---

## 6. Election Algorithms

**Goal:** choose **one** process as coordinator after startup or after the coordinator fails. Every process has a unique ID, and the **highest ID wins**. All processes must agree on the result.

### 6.1 Bully Algorithm

Assumes every process knows the IDs of all others, and links are reliable.

**Steps:**
1. A process P notices the coordinator is dead and sends **ELECTION** to all processes with **higher IDs**.
2. If **nobody replies**, P wins and sends **COORDINATOR** to everyone.
3. If a higher process replies **OK**, P stops, and the higher process starts its own election.
4. The highest alive process eventually wins ("bullies" the others).

**Simulation output:**
```
Alive=[1, 2, 3, 4] Coordinator (P5) crashed. P2 detects.
P2 sends ELECTION to ['P3', 'P4']
['P3', 'P4'] reply OK -> P4 takes over
P4 sends ELECTION to nobody
P4 broadcasts COORDINATOR. New leader = P4
```

```
 P2 --ELECTION--> P3, P4
 P3, P4 --OK--> P2
 P3 --ELECTION--> P4     (P4 replies OK)
 P4 --COORDINATOR--> P1, P2, P3
```

- **Messages:** worst case O(N²) (lowest process starts), best case N−1
- **Drawback:** high message cost, and a restarted high-ID process can take over again

### 6.2 Ring Algorithm

Processes are in a **logical ring**. No coordinator ID needs to be known by everyone, only the next neighbor.

**Steps:**
1. A process that notices the failure creates an **ELECTION** message containing its own ID and sends it to its next **live** neighbor, skipping dead ones.
2. Each process that receives it **appends its own ID** and forwards it.
3. When the message returns to the initiator, it picks the **highest ID** from the list.
4. A **COORDINATOR** message circulates once around the ring announcing the winner.

**Simulation output** (ring of IDs [3, 7, 1, 9, 4], starter id=1):
```
-> node id=9: list=[1, 9]
-> node id=4: list=[1, 9, 4]
-> node id=3: list=[1, 9, 4, 3]
-> node id=7: list=[1, 9, 4, 3, 7]
Message back at starter. Leader = max = 9
```

- **Messages:** about 2N (N for election plus N for coordinator announcement)
- **Variant, Chang-Roberts:** a node forwards only if the incoming ID is larger than its own. Average O(N log N), worst case O(N²).

### 6.3 Comparison

| Feature | Bully | Ring |
|---|---|---|
| Topology | Fully connected | Logical ring |
| Who gets contacted | Higher-ID processes | Next neighbor |
| Messages (worst) | O(N²) | O(N) per round (about 2N) |
| Speed | Faster when high IDs are alive | Slower, sequential |
| Needs all IDs known | Yes | No (only next node) |

**Other election algorithms (mention in an exam):** Chang-Roberts, Hirschberg-Sinclair (O(N log N)), and **Raft/Paxos leader election** in modern systems. ZooKeeper uses ephemeral sequential nodes for election.

---

## 7. Mutual Exclusion

**Problem:** ensure only **one process at a time** is in the **critical section (CS)**, using only message passing.

### 7.1 Requirements
1. **Safety:** at most one process in the CS at once
2. **Liveness:** no deadlock or starvation. Every request is eventually granted.
3. **Fairness (ordering):** requests are served in the order they were made (using logical timestamps)

### 7.2 Performance Metrics
- **Messages** per CS entry
- **Synchronization delay:** time between one process leaving and the next entering
- **Response time:** from request to leaving the CS
- **Throughput:** CS executions per unit time

### 7.3 Classification

```
 Mutual Exclusion
 ├── Permission-based (ask others)
 │    ├── Centralized
 │    ├── Lamport
 │    ├── Ricart-Agrawala
 │    └── Maekawa (quorum)
 └── Token-based (hold a unique token)
      ├── Token Ring
      ├── Suzuki-Kasami
      └── Raymond (tree)
```

### 7.4 Centralized Algorithm
One **coordinator** grants permission.
1. Process sends **REQUEST** to the coordinator.
2. If the CS is free, the coordinator sends **GRANT**. Otherwise it queues the request.
3. On leaving, the process sends **RELEASE**, and the coordinator grants the next in the queue.

- **Messages:** 3 per entry (request, grant, release)
- **Pros:** simple, fair, no starvation
- **Cons:** single point of failure and a bottleneck. A dead coordinator is indistinguishable from "permission denied."

### 7.5 Lamport's Algorithm
Every process keeps a **request queue** ordered by (timestamp, process ID).
1. To enter, Pi **broadcasts REQUEST(ts, i)** and adds it to its own queue.
2. On receiving a REQUEST, others add it to their queue and send **REPLY**.
3. Pi enters the CS when (a) its request is at the **head of its own queue** and (b) it has received a **message with a larger timestamp from all** others.
4. On exit, Pi **broadcasts RELEASE**. Everyone removes its request.

**Messages:** **3(N−1)** per entry: (N−1) REQUEST + (N−1) REPLY + (N−1) RELEASE.

### 7.6 Ricart-Agrawala Algorithm
An optimization of Lamport's: it **removes the RELEASE message** by deferring replies.

1. To enter, Pi **multicasts REQUEST(ts, i)** to all others.
2. On receiving a REQUEST, Pj:
   - If Pj is **not interested** in the CS: sends **REPLY** immediately
   - If Pj is **in the CS**: **defers** the reply
   - If Pj **also wants** the CS: compares timestamps. The **lower (ts, id)** wins. If the sender is lower, reply. Otherwise defer.
3. Pi enters the CS after receiving **REPLY from all N−1** processes.
4. On exit, Pi sends all **deferred REPLYs**.

**Simulation output:**
```
Requests (ts,id): {'P1': (5, 1), 'P2': (3, 2), 'P3': (8, 3)}
P2 enters CS after replies from all (N-1=2 REPLYs)
P1 enters CS ...
P3 enters CS ...
Messages per CS entry = 2(N-1) = 4
```
The entry order follows timestamps (3, then 5, then 8), so requests are served in logical-time order.

- **Messages:** **2(N−1)** per entry
- **Drawback:** if any process crashes, it never replies, so the system **blocks**. All N processes are involved.

### 7.7 Maekawa's Algorithm (Quorum-based)
Each process only needs permission from a **subset (quorum)** of size about **√N**, and any two quorums **overlap** in at least one process, which guarantees mutual exclusion.

- **Messages:** about **3√N** (request, reply, release) with no deadlocks. It can deadlock in the basic version and needs extra messages (FAILED, INQUIRE, YIELD) to fix that.
- Each process votes for only one request at a time.

### 7.8 Token Ring Algorithm
A single **token** circulates around a logical ring. Only the holder may enter the CS.

1. If a process wants the CS, it waits for the token.
2. After using the CS, it passes the token on. If it doesn't want the CS, it passes the token immediately.

- **Messages:** 1 to N−1 per entry (can be wasted while the token circulates with nobody waiting)
- **Pros:** no starvation, simple
- **Cons:** **lost token** (needs detection and regeneration), and a crashed node breaks the ring

### 7.9 Suzuki-Kasami (Broadcast token)
- A requesting process **broadcasts REQUEST(i, sn)** (sequence number) to all others.
- The token holder sends the token to the requester once it is idle.
- The token holds a **queue** and an array `LN[]` of last-served request numbers.
- **Messages:** N per entry (N−1 requests plus 1 token), or 0 if the process already holds the token.

### 7.10 Raymond's Tree Algorithm
Processes are arranged in a **tree**, and the token moves along tree edges. Requests travel toward the token holder.
- **Messages:** **O(log N)** on average.

### 7.11 Comparison of Algorithms

**Simulation output (messages per CS entry):**
```
N=5   Centralized=3  Ricart-Agrawala=8    Lamport=12   Token-ring=1..5
N=10  Centralized=3  Ricart-Agrawala=18   Lamport=27   Token-ring=1..10
N=50  Centralized=3  Ricart-Agrawala=98   Lamport=147  Token-ring=1..50
```

| Algorithm | Messages / entry | Sync delay | Problem |
|---|---|---|---|
| Centralized | 3 | 2 messages | Coordinator failure |
| Lamport | 3(N−1) | 1 message | Many messages, any crash blocks |
| Ricart-Agrawala | 2(N−1) | 1 message | Any crash blocks |
| Maekawa | ~3√N | 2 messages | Deadlock-prone |
| Token ring | 1 to N−1 | 1 to N−1 | Lost token, ring break |
| Suzuki-Kasami | N (or 0) | 1 message (token) | Lost token |
| Raymond | O(log N) | O(log N) | Tree failure |

---

## 8. Likely Interview Questions

1. **Why can't we use physical clocks to order events in a distributed system?**
   Clocks drift and have skew, and message delay is unpredictable, so timestamps from different machines are not comparable.

2. **Cristian vs Berkeley?**
   Cristian syncs to an external time server (UTC) using RTT/2. Berkeley averages all clocks and sends adjustments, so it needs no UTC source.

3. **What is the happened-before relation?**
   Order within a process, send before receive, and transitivity. Events not related by it are concurrent.

4. **Lamport vs vector clocks?**
   Lamport: one integer, a→b ⇒ C(a)<C(b) only. Vector: N integers, both directions hold, and it detects concurrency.

5. **What is a consistent cut?**
   A cut where no message crosses from the future into the past, meaning every receive in the cut has its send in the cut.

6. **Explain Chandy-Lamport.**
   Marker messages trigger processes to record local state and incoming channel contents, giving a consistent snapshot without stopping the system. It needs FIFO channels.

7. **Bully vs ring election?**
   Bully contacts higher IDs, with O(N²) worst case. Ring passes a list around, with about 2N messages. The highest ID wins in both.

8. **Which mutual exclusion algorithm needs the fewest messages?**
   Centralized needs 3. Raymond gives O(log N) and Maekawa gives O(√N) for distributed ones.

9. **Why does Ricart-Agrawala need 2(N−1) messages?**
   (N−1) requests plus (N−1) replies, with no release message because replies are deferred.

10. **What are the disadvantages of Ricart-Agrawala?**
    A failed process never replies, so the system blocks. It also involves all N nodes in every entry.

11. **Permission-based vs token-based mutual exclusion?**
    Permission-based asks others for consent using timestamps. Token-based gives the right to enter to whoever holds a unique token, and the main risk is token loss.

12. **How does Google Spanner handle time?**
    TrueTime returns an uncertainty interval, and the system waits out the uncertainty before committing, which gives external consistency.

13. **Does Lamport's total ordering give fairness?**
    Yes. Requests are served in timestamp order, with the process ID breaking ties.

---

## 9. Quick Revision Sheet

- **Clock sync:** Cristian = Ts + RTT/2. Berkeley = average, send the adjustment. NTP = strata hierarchy, θ = ((T2−T1)+(T3−T4))/2.
- **Never set a clock backward**, only slow it down.
- **Happened-before (→):** same process order, send→receive, transitive. Otherwise concurrent.
- **Lamport clock:** increment on events, `max(own, ts)+1` on receive. One-directional implication.
- **Vector clock:** N counters, element-wise max on receive. Detects concurrency.
- **Consistent cut:** no receive without its send. **Chandy-Lamport:** markers, FIFO channels, records local state plus channel state.
- **Election:** Bully (higher IDs, O(N²), highest alive wins). Ring (list circulates, about 2N messages).
- **Mutual exclusion requirements:** safety, liveness, fairness.
- **Messages:** Centralized 3, Lamport 3(N−1), Ricart-Agrawala 2(N−1), Maekawa ~3√N, Suzuki-Kasami N, Raymond log N, Token ring 1..N.
- **Failure weakness:** Ricart-Agrawala and Lamport block on a crash. Token schemes suffer from token loss.

