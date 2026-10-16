# Distributed Systems: Unit 4 Notes (Recovery and Fault Tolerance)

---

## 1. Failures and Their Classification

### 1.1 Fault, Error, Failure

```
 FAULT  --(activates)-->  ERROR  --(propagates)-->  FAILURE
 (cause: bug, bad disk)   (wrong internal state)    (service deviates from spec)
```

A system is **fault tolerant** if it keeps providing correct service **despite faults**.

### 1.2 Dependability Attributes

| Attribute | Meaning |
|---|---|
| **Availability** | Probability the system is up at a given instant |
| **Reliability** | Continuity of correct service over a time interval |
| **Safety** | No catastrophic consequence when it fails |
| **Maintainability** | How easily it can be repaired |

$$\text{Availability} = \frac{\text{MTTF}}{\text{MTTF}+\text{MTTR}} \qquad \text{MTBF} = \text{MTTF}+\text{MTTR} \qquad R(t) = e^{-\lambda t},\ \lambda=\tfrac{1}{\text{MTTF}}$$

**Simulation output:**
```
MTTF=1000h MTTR=10h -> MTBF=1010h, Availability=99.0099%
Reliability R(100h)=0.905    R(1000h)=0.368
Replication (single node availability a=0.99):
  1 replica: 0.99000000     2 replicas: 0.99990000
  3 replicas: 0.99999900    4 replicas: 0.99999999
```
Replicas improve availability as **A = 1 − (1−a)ⁿ**. Two 99% nodes give 99.99%.

### 1.3 Classification by Duration

| Type | Description | Example |
|---|---|---|
| **Transient** | Occurs once and disappears | Cosmic-ray bit flip, a lost packet |
| **Intermittent** | Appears, vanishes, reappears | Loose connector, a race condition |
| **Permanent** | Persists until repaired | Burnt-out disk, software bug |

### 1.4 Classification by Behavior (Failure Models)

| Failure type | Description |
|---|---|
| **Crash** | Server halts, but was working correctly until it halted |
| **Omission** | Fails to respond. **Receive omission:** does not receive a message. **Send omission:** does not send one. |
| **Timing** | Responds outside the specified time interval |
| **Response failure** | **Value failure:** wrong value. **State-transition failure:** deviates from the correct flow of control |
| **Arbitrary (Byzantine)** | May produce any output at any time, including malicious or conflicting outputs |

Severity: **Crash < Omission < Timing < Response < Byzantine**.

**Related terms:**
- **Fail-stop:** crash that other processes can reliably detect
- **Fail-silent:** crash or omission, but others cannot tell it from slowness
- **Fail-safe:** if it fails, the failure is harmless (arbitrary but benign)

### 1.5 Classification by Component

| Component | Failures |
|---|---|
| **Process / site** | Crash, hang, software bug |
| **Communication** | Message loss, duplication, reordering, corruption, **network partition** |
| **Storage** | Disk crash, bit rot (media failure) |
| **Transaction** | Abort due to deadlock, constraint violation |

### 1.6 Techniques to Mask Failures (Redundancy)
- **Information redundancy:** error-correcting codes, checksums, parity
- **Time redundancy:** repeat the operation (retransmit, retry a transaction)
- **Physical redundancy:** extra hardware or process replicas (triple modular redundancy)

---

## 2. Checkpoints and Rollback Recovery

### 2.1 Stable Storage
Storage that **survives crashes**, built from mirrored disks, RAID, or careful write protocols. Checkpoints and logs are written here.

### 2.2 Backward vs Forward Recovery

| Backward error recovery | Forward error recovery |
|---|---|
| Restore a **previous correct state** (checkpoint) and retry | Move to a **new correct state** by correcting the error |
| General purpose | Needs knowledge of the error (e.g., error-correcting codes) |
| Cost: lost work, checkpoint overhead | Cost: application-specific |

Checkpointing implements **backward recovery**.

### 2.3 Consistent Global State and Orphan Messages
A set of local checkpoints is a **consistent recovery line** if there is no orphan message across it.

| Message type | Definition | Problem |
|---|---|---|
| **Orphan** | Receive recorded in the checkpoint, **send not recorded** | The receiver reflects an event that never happened, so it is **inconsistent** |
| **Lost (in-transit)** | Send recorded, receive not recorded | Can be fixed by logging or retransmitting |

### 2.4 Uncoordinated (Independent) Checkpointing
Each process checkpoints on its own schedule, with no coordination.

- **Pros:** simple, no checkpoint-time overhead, processes choose convenient moments
- **Cons:** **domino effect**, many checkpoints to store, garbage collection is hard

**Domino effect:** after a failure, a rollback makes some message an orphan, forcing the receiver to roll back, which creates another orphan, and so on, possibly back to the **initial state**.

**Simulation** (P1 fails at t=10; P1 checkpoints at 0,2,4,6,8; P2 at 0,3,5,7):
```
P1 fails at t=10 -> rolls back to its last checkpoint: 8
step 1: msg a (P1->P2) ORPHAN -> P2 rolls back to ckpt t=7
step 2: msg b (P2->P1) ORPHAN -> P1 rolls back to ckpt t=6
step 3: msg c (P1->P2) ORPHAN -> P2 rolls back to ckpt t=5
step 4: msg d (P2->P1) ORPHAN -> P1 rolls back to ckpt t=4
step 5: msg e (P1->P2) ORPHAN -> P2 rolls back to ckpt t=3
step 6: msg f (P2->P1) ORPHAN -> P1 rolls back to ckpt t=2
step 7: msg g (P1->P2) ORPHAN -> P2 rolls back to ckpt t=0
step 8: msg h (P2->P1) ORPHAN -> P1 rolls back to ckpt t=0
Final recovery line: {P1: 0, P2: 0}  => domino effect
```

```
 P1: |0---|2---|4---|6---|8--- fail
 P2: |0------|3---|5---|7---
        ^ messages criss-cross between checkpoints, so every
          rollback orphans a message and forces another rollback
```

### 2.5 Coordinated Checkpointing
All processes take checkpoints **together**, so the set is always a **consistent cut**.

```
 Coordinated checkpoint at t=7.9, P1 fails at t=10
 -> both roll back to the t=7.9 line only. No cascading, 2.1 time units lost.
```

| Variant | How it works |
|---|---|
| **Blocking (2-phase, Koo-Toueg)** | Phase 1: the initiator asks processes to take **tentative** checkpoints (and stop sending application messages). Phase 2: if all succeed, tentative checkpoints become **permanent**. |
| **Non-blocking (Chandy-Lamport)** | Marker messages trigger checkpoints while the application keeps running (see Unit 2) |
| **Minimal (Koo-Toueg)** | Only processes that **communicated with the initiator since the last checkpoint** participate, which reduces overhead |

- **Pros:** no domino effect, only the **latest** checkpoint is kept, simple recovery
- **Cons:** coordination messages, a failure during checkpointing can abort it, blocking variants stall the application

### 2.6 Communication-Induced Checkpointing
Processes take **local** checkpoints plus **forced** checkpoints triggered by information **piggybacked** on application messages, which avoids useless checkpoints and the domino effect without explicit coordination messages.

### 2.7 Message Logging (Log-Based Rollback Recovery)
Assumes **piecewise deterministic** execution. If non-deterministic events (message receipts) are logged, a process can **replay** them after restoring a checkpoint.

| Protocol | When messages are logged | Orphans | Overhead |
|---|---|---|---|
| **Pessimistic** | **Before** delivery (synchronously) | None | High failure-free cost, simple recovery |
| **Optimistic** | Asynchronously, in batches | Possible, so survivors may roll back | Low |
| **Causal** | Info piggybacked on messages, stored in senders | None | Medium |

### 2.8 Checkpoint Interval
Too frequent means high overhead. Too rare means a lot of lost work. **Young's approximation:**

$$T_{opt} \approx \sqrt{2 \cdot C \cdot \text{MTBF}}$$

*Example:* checkpoint cost C = 10 s, MTBF = 8 h = 28,800 s, so T ≈ √(2·10·28800) ≈ **759 s ≈ 12.6 minutes**.

---

## 3. Recovery in Distributed Databases

### 3.1 Failure Types in a DDBMS
- **Transaction failure** (abort, deadlock)
- **Site (system) failure:** loses volatile memory
- **Media failure:** disk lost
- **Communication failure** and **network partition**

### 3.2 Logging and Write-Ahead Logging (WAL)
**WAL rule:** a log record describing an update must reach **stable storage before** the updated data does.

| Log record | Meaning |
|---|---|
| `<T, start>` | Transaction began |
| `<T, X, old, new>` | Update record (undo and redo info) |
| `<T, commit>` / `<T, abort>` | Outcome |
| `<checkpoint>` | Limits how far back recovery must scan |

### 3.3 Undo and Redo

| Update policy | Recovery needs |
|---|---|
| **Deferred update** (write to DB only after commit) | **Redo** only |
| **Immediate update** (write before commit) | **Undo and redo** |

**Restart after a site crash:**
- Transactions with `<commit>` in the log: **REDO**
- Transactions with no `<commit>`: **UNDO**
- **Checkpoints** bound the log scan. (ARIES: analysis, redo, undo passes.)

### 3.4 Added Problems in a Distributed Setting
- A transaction runs at **several sites**, so all sites must reach the **same** outcome (see section 7)
- A recovering site must learn the outcome of **in-doubt** transactions from other sites
- Sites may be unreachable, so **partitions** need a policy

### 3.5 Replicated Data and Partitions
**Quorum consensus** for N copies:
- **Read quorum Vr** and **write quorum Vw** must satisfy **Vr + Vw > N** (read and write overlap) and **Vw > N/2** (two writes overlap)
- Example, N = 5: Vr = 3, Vw = 3.
- Only the **majority partition** can write, so there is **no split-brain**.

| Replica control | Idea | Trade-off |
|---|---|---|
| **ROWA** (read one, write all) | Fast reads, writes need all sites | Write blocked if any site down |
| **Primary copy** | One copy coordinates updates | Primary is a bottleneck, needs failover |
| **Quorum** | Majority read/write | Balanced availability |

---

## 4. Process Resilience

### 4.1 Process Groups
Replicate processes into a **group** so that if one fails, another takes over. The group appears as a **single logical process**.

| Flat group | Hierarchical group |
|---|---|
| All members equal, decisions by voting | One **coordinator**, others are workers |
| No single point of failure | Coordinator is a single point of failure |
| Slower, more complex decisions | Faster decisions, simple |

**Group membership:** needs join and leave operations and detection of crashed members, which must be consistent for all members.

### 4.2 Replication Styles

| Style | Description | Pros and cons |
|---|---|---|
| **Primary-backup (passive)** | Primary handles requests and updates backups. On failure a backup is promoted. | Cheap, but failover delay |
| **Active replication (state machine)** | **All replicas** execute every request in the **same order** | Instant masking, needs **deterministic** execution and total-order multicast |

### 4.3 K-Fault Tolerance
A group is **k fault tolerant** if it continues to work despite **k** faulty members.

| Failure model | Replicas needed |
|---|---|
| **Crash / fail-silent** | **k + 1** (one surviving member suffices) |
| **Byzantine (masking by majority vote)** | **2k + 1** (correct members outvote the bad ones) |
| **Byzantine (reaching agreement)** | **3k + 1** |

**Simulation output (voting):**
```
Majority voting, 3 replicas, 1 Byzantine (k=1, 2k+1=3):
 replica outputs: [42, 42, 17] -> voted result: 42
```

### 4.4 Byzantine Agreement (Byzantine Generals)
**Problem:** n generals, up to k traitors. All **loyal** generals must agree on the same plan, and if the commander is loyal they must follow his order.

**Result:** agreement is possible only if **n ≥ 3k + 1**.

**Simulation output (one round of relaying, OM(1)):**
```
n=3 (loyal commander, 1 traitor lieutenant): {'L1': 'TIE'}      -> cannot decide
n=4 (loyal commander, traitor L3):  {'L1': 1, 'L2': 1}          -> loyal ones agree on 1
n=4 (TRAITOR commander):            {'L1': 1, 'L2': 1, 'L3': 1} -> all agree
n >= 3k+1 needed to reach agreement with k Byzantine faults
```
With n = 3 and one traitor, the honest lieutenant gets one value from the commander and a conflicting relayed value, which is a tie. With n = 4 the majority resolves it.

### 4.5 Consensus
All non-faulty processes must **agree** on one value, which must be one of the **proposed** values. Properties: **agreement, validity, termination**.
- **FLP result:** in a **purely asynchronous** system, no deterministic algorithm can guarantee consensus if even one process may crash.
- Practical systems (**Paxos, Raft**) use **timeouts and leader election**, and sacrifice liveness (not safety) in bad periods.

### 4.6 Failure Detection
- **Heartbeats:** processes periodically say "I'm alive"
- **Ping / probe:** a monitor asks and waits for a reply
- **Gossip** and **phi-accrual** detectors give a suspicion level instead of yes/no
- In an asynchronous system, a timeout cannot distinguish a **crashed** process from a **slow** one, so detectors are **unreliable**.

---

## 5. Reliable Client-Server Communication

### 5.1 Point-to-Point Communication
**TCP** masks **omission failures** (lost, duplicate, out-of-order packets) using acknowledgments, sequence numbers, and retransmission. It does **not** mask **crashes**: if the connection breaks, the application must handle it.

### 5.2 Five RPC Failure Classes

| # | Failure | Remedy |
|---|---|---|
| 1 | **Client cannot locate the server** | Raise an exception (hurts transparency) |
| 2 | **Request message lost** | Client **timeout** and **retransmit**. If all are lost, treat as class 1. |
| 3 | **Server crashes after receiving the request** | Depends on semantics (5.3) |
| 4 | **Reply message lost** | Retransmit. Non-idempotent operations need **duplicate detection**. |
| 5 | **Client crashes after sending** | **Orphan** computation (5.4) |

**Class 4 detail:** if the operation is **idempotent** (read), just retry. If not (transfer money), the client puts a **sequence number** in each request, and the server keeps a **reply cache** and filters duplicates, which gives **at-most-once** semantics.

### 5.3 Server Crashes: Why Retrying Is Not Enough
The server must perform an action (print) and send a completion message (ACK). It can do **M then P** or **P then M**, and crash at different points. The client can reissue **always**, **never**, **only when ACKed**, or **only when not ACKed**.

**Simulation output** (OK = executed once, DUP = twice, ZERO = never):
```
Server order M->P   (M=send ACK, P=print, C=crash)
client strategy MPC     MC(P)   C(MP)
always          DUP     OK      OK
never           OK      ZERO    ZERO
when ACKed      DUP     OK      ZERO
when not ACKed  OK      ZERO    OK

Server order P->M
client strategy PMC     PC(M)   C(PM)
always          DUP     DUP     OK
never           OK      OK      ZERO
when ACKed      DUP     OK      ZERO
when not ACKed  OK      DUP     OK
=> NO strategy gives OK in every column
```
The client **cannot tell** whether the server crashed before or after doing the work, so **exactly-once cannot be achieved by a retry policy alone**. It needs **idempotent operations**, or **transactions/state logging at the server** (stateful, at-most-once, or idempotency keys).

**RPC semantics** (from Unit 1): maybe, at-least-once, at-most-once, exactly-once (approximated).

### 5.4 Orphans (Client Crashes)
An **orphan** is a server-side computation whose client has died. It wastes CPU and may hold locks.

| Solution | How |
|---|---|
| **Extermination** | Client logs every RPC before making it. On reboot, it reads the log and kills orphans. (Costly: logging, and orphans can spawn grand-orphans.) |
| **Reincarnation** | On reboot, the client broadcasts a new **epoch** number. Servers kill computations from older epochs. |
| **Gentle reincarnation** | On receiving an epoch broadcast, servers kill a computation only if its **owner cannot be found**. |
| **Expiration** | Each RPC gets a time quantum **T**. It must finish or ask for more time. After a crash, wait T before restarting. |

---

## 6. Reliable Group Communication

### 6.1 Goal and Difficulty
**Reliable multicast:** a message sent to a group is delivered to **all non-faulty members**. It is easy with no failures and a fixed group, and hard because of **scalability**, **failures during multicast**, and **changing membership**.

### 6.2 Basic Reliable Multicast
The sender keeps a **history buffer**, numbers each message, and receivers detect missing messages.

| ACK-based | NACK-based |
|---|---|
| Receiver acknowledges **every** message | Receiver reports only **missing** messages (sequence gaps) |
| Sender knows exactly who has what | Sender never knows if everyone got it (must keep history) |
| **ACK implosion:** R receivers send R ACKs | Little feedback in the loss-free case |

**Simulation output (NACK-based recovery):**
```
msg 3 sent -> LOST at receiver
msg 4 sent -> LOST at receiver
msg 5 received (gap detected! expected 3) -> NACK[3, 4]
sender retransmits 3, 4 -> received
all delivered in order: [1, 2, 3, 4, 5, 6]

Feedback load for one multicast to R receivers, no loss:
 R=10     ACK-based: 10 ACKs   | NACK-based: 0
 R=100000 ACK-based: 100000    | NACK-based: 0
```
The sim re-sends the NACK when msg 6 arrives. Real protocols use timers to avoid these duplicate NACKs.

### 6.3 Scalable Reliable Multicast (SRM, NACK suppression)
- A receiver that detects a loss waits a **random time**, then **multicasts** its NACK to the whole group.
- Other receivers who need the same message **suppress** their own NACK when they hear one.
- **Any** receiver that has the message may answer (also after a random delay), so retransmission is also shared.
- Reduces feedback, but all nodes see all NACKs and timers need tuning.

### 6.4 Hierarchical Feedback Control
Receivers are organized in a **tree** of subgroups. Each subgroup has a **local coordinator** that handles its members' ACKs/NACKs and retransmits locally, and only coordinators talk to the sender.
- **Pros:** very scalable
- **Cons:** building and maintaining the tree under changing membership is hard

### 6.5 Atomic Multicast
A message is delivered to **all** members of the group or to **none** (all non-faulty members agree), even if the sender crashes mid-multicast. It requires agreement on the **group membership** at the time of sending, and is usually combined with an **ordering** guarantee (FIFO, causal, total, from Unit 1).

### 6.6 Virtual Synchrony
A model that makes multicast appear to occur at a **well-defined point** relative to **membership changes**.

- A **view** is the current list of group members.
- Messages are delivered **within a view**. A **view change** acts as a **barrier**: all messages multicast in the old view are delivered to all surviving members before the new view is installed.
- If the sender crashes mid-multicast, either **all surviving** members get the message or **none** do.

```
 View V1 = {P1,P2,P3,P4}               View V2 = {P1,P2,P3}
 P4 --m--> (reaches P1, P2 only)        P4 crashes during multicast
 FLUSH: P1/P2 forward m to P3  ------>  all survivors have m (or none do)
 ... then install V2
```

**Implementation (flush protocol):** on a view change, every member forwards unstable messages and sends a **flush** message. When flushes arrive from all survivors, the new view is installed.

### 6.7 Message Ordering Summary

| Ordering | Guarantee | Cost |
|---|---|---|
| Unordered | None | Lowest |
| **FIFO** | Same-sender messages in send order | Low |
| **Causal** | Cause delivered before effect | Vector clocks |
| **Total (atomic)** | Everyone sees the **same order** | Sequencer or consensus |

---

## 7. Distributed Commit and Recovery

### 7.1 The Problem
A distributed transaction touches several sites. **Atomicity** demands that **all** sites commit or **all** abort, even if some crash or messages are lost. The **atomic commit protocol** achieves this, with one **coordinator** and several **participants**.

### 7.2 One-Phase Commit
The coordinator just tells everyone to commit. **Flaw:** a participant that cannot commit (constraint violation, local failure) has no way to say no.

### 7.3 Two-Phase Commit (2PC)

**Phase 1 (voting):**
1. Coordinator sends **VOTE-REQUEST** to all participants.
2. Each participant that can commit **forces `<ready>` to its log**, then replies **VOTE-COMMIT**. Otherwise it aborts locally and replies **VOTE-ABORT**.

**Phase 2 (decision):**
3. If **all** voted commit, the coordinator logs and sends **GLOBAL-COMMIT**. If **any** voted abort (or timed out), it sends **GLOBAL-ABORT**.
4. Participants act on the decision and send **ACK**.

```
 Coordinator              Participants
     |--- VOTE-REQUEST --->|
     |<--- VOTE-COMMIT ----|     (participant is now READY, uncertain)
     |--- GLOBAL-COMMIT -->|
     |<------- ACK --------|
```

**State machines:**
```
 Coordinator:  INIT -> WAIT -> COMMIT | ABORT
 Participant:  INIT -> READY -> COMMIT | ABORT
```

**Simulation output:**
```
votes = {P1: YES, P2: YES, P3: YES}  -> global COMMIT
votes = {P1: YES, P2: NO,  P3: YES}  -> global ABORT
```

**Timeouts:**

| Who / state | Timeout action |
|---|---|
| Coordinator in **WAIT** | Abort and send GLOBAL-ABORT |
| Participant in **INIT** | Abort unilaterally |
| Participant in **READY** | **Cannot decide alone**, runs the **termination protocol** |

**Termination protocol (participant P in READY contacts others):**

| Other participant Q is in state | P's action |
|---|---|
| **COMMIT** | Commit |
| **ABORT** | Abort |
| **INIT** (never voted) | Abort (Q will abort too) |
| **READY** | Cannot decide, ask others. If **all** are READY, **blocked** |

**Simulation output (coordinator crashes after collecting votes, before sending a decision):**
```
2PC, all READY :  BLOCKED (all READY/uncertain; must wait for coordinator to recover)
2PC, P1 got COMMIT before crash:  COMMIT (someone already committed)
```

**2PC's weakness:** it is a **blocking protocol**. If the coordinator fails while all participants are READY, they hold locks and wait until the coordinator recovers.

**Cost:** 3N messages (vote request, votes, decision), or **4N with ACKs**. Two rounds.

### 7.4 Three-Phase Commit (3PC)
Adds a **PRECOMMIT** state between READY and COMMIT, so no state is adjacent to both COMMIT and ABORT.

```
 Coordinator:  INIT -> WAIT -> PRECOMMIT -> COMMIT | ABORT
 Participant:  INIT -> READY -> PRECOMMIT -> COMMIT | ABORT
```

**Phases:** (1) CAN-COMMIT / vote, (2) PRECOMMIT / ACK, (3) DO-COMMIT / ACK.

**Why it is non-blocking:** reaching PRECOMMIT means **all** participants voted yes, so survivors can decide using this rule:

| Survivor states | Decision |
|---|---|
| Any **COMMIT** | Commit |
| Any **ABORT** | Abort |
| Any **PRECOMMIT** | **Commit** (all voted yes) |
| All **READY** | **Abort** (nobody is PRECOMMIT, so no one can have committed) |

**Simulation output:**
```
3PC, all READY :     ABORT (nobody reached PRECOMMIT => safe to abort, NON-blocking)
3PC, P1 PRECOMMIT:   COMMIT (a participant reached PRECOMMIT => all voted YES)
```

**Cost:**
```
N=3   2PC = 4N = 12   3PC = 6N = 18
N=5   2PC = 4N = 20   3PC = 6N = 30
N=10  2PC = 4N = 40   3PC = 6N = 60
```

**Limits of 3PC:** assumes **no network partitions** and **bounded delays**. It is not widely used in practice. Modern systems run **2PC over replicated, consensus-backed participants** (Paxos Commit, Google Spanner) or use **Sagas** with compensating actions.

### 7.5 Comparison

| Feature | 2PC | 3PC |
|---|---|---|
| Phases | 2 | 3 |
| Messages (with ACKs) | 4N | 6N |
| Blocking on coordinator crash | **Yes** | **No** (crash failures only) |
| Network partitions | Blocks, but stays safe | **Can violate safety** |
| Latency | Lower | Higher |
| Used in practice | Very common | Rare |

### 7.6 Recovery After Crashes (Logging Rules)

**Logging requirements:**
- The coordinator **forces** `<begin-commit>` and then the **decision** (`<commit>` or `<abort>`) to stable storage **before** sending the decision.
- A participant **forces `<ready>` before** sending VOTE-COMMIT.

**Participant recovery (after restart):**

| Log contains | Action |
|---|---|
| `<commit>` | **Redo** |
| `<abort>` | **Undo** |
| **No `<ready>`** and no decision | Unilaterally **abort** (undo), since it never voted yes |
| `<ready>` but no decision | **Uncertain:** ask the coordinator or other participants (termination protocol) |

**Coordinator recovery:**

| Log contains | Action |
|---|---|
| `<commit>` | Resend GLOBAL-COMMIT to participants who have not ACKed |
| `<abort>` | Resend GLOBAL-ABORT |
| `<begin-commit>` but **no decision** | **Abort** and send GLOBAL-ABORT |
| Nothing | Transaction never started the protocol, so abort |

**Presumed abort:** if the coordinator has no record of a transaction, it answers "abort", which saves forced writes on aborts.

---

## 8. Likely Interview Questions

1. **What are the types of failures in a distributed system?**
   Crash, omission, timing, response, and arbitrary (Byzantine), classified by duration as transient, intermittent, or permanent.

2. **Fail-stop vs fail-silent?**
   Fail-stop: crash that is reliably detectable. Fail-silent: crash or omission that looks the same as slowness.

3. **What is the domino effect?**
   Uncoordinated checkpoints cause rollback to orphan messages, which forces further rollbacks, possibly to the initial state.

4. **What is an orphan message?**
   A message whose receive is in the saved state but whose send is not, so the state is inconsistent.

5. **Coordinated vs uncoordinated checkpointing?**
   Coordinated yields a consistent line with no domino effect but needs coordination. Uncoordinated is simple but can cascade and stores many checkpoints.

6. **Pessimistic vs optimistic message logging?**
   Pessimistic logs before delivery, so no orphans but slower. Optimistic logs asynchronously, faster, but orphans can force survivors to roll back.

7. **What is write-ahead logging?**
   The log record must reach stable storage before the data update.

8. **How many replicas to tolerate k faults?**
   k+1 for crash faults, 2k+1 for masking Byzantine faults by voting, 3k+1 to reach Byzantine agreement.

9. **Why can't RPC guarantee exactly-once?**
   The client cannot distinguish whether the server crashed before or after performing the action. Retry policies alone always have a failing case (shown in the table).

10. **How do you handle orphans?**
    Extermination, reincarnation, gentle reincarnation, expiration.

11. **ACK implosion and how do you avoid it?**
    Too many ACKs at the sender in reliable multicast. Use NACKs, feedback suppression (SRM), or hierarchical feedback.

12. **What is virtual synchrony?**
    Group views plus a flush at view changes, so messages sent in a view are delivered to all survivors before the view changes.

13. **Explain 2PC. Why can it block?**
    Vote phase then decision phase. If the coordinator crashes while participants are READY, they cannot decide alone and must wait.

14. **How does 3PC remove blocking?**
    The PRECOMMIT state tells survivors that everyone voted yes, so they can decide without the coordinator. It requires no partitions and bounded delays.

15. **What does a participant do after a crash in the READY state?**
    Contacts the coordinator or other participants (termination protocol) to learn the outcome.

16. **Read/write quorum rules?**
    Vr + Vw > N and Vw > N/2.

---

## 9. Quick Revision Sheet

- **Fault → error → failure.** Failure types: crash < omission < timing < response < Byzantine.
- **Availability = MTTF/(MTTF+MTTR).** Replication: A = 1 − (1−a)ⁿ.
- **Redundancy:** information, time, physical.
- **Checkpointing = backward recovery.** Uncoordinated has **domino effect**. Coordinated (Koo-Toueg, Chandy-Lamport) has a consistent cut and no cascade. Communication-induced uses forced checkpoints.
- **Orphan:** receive recorded, send not recorded. **Lost message:** send recorded, receive not.
- **Message logging:** pessimistic (no orphans), optimistic (orphans possible), causal (piggybacked).
- **Young's formula:** T ≈ √(2·C·MTBF).
- **DB recovery:** WAL, redo for committed, undo for uncommitted, checkpoints bound the log scan. Quorum: Vr+Vw>N, Vw>N/2.
- **Replicas needed:** k+1 (crash), 2k+1 (Byzantine masking), **3k+1** (Byzantine agreement). **FLP:** no deterministic consensus in an async system with a crash.
- **RPC failures:** can't locate server, lost request, server crash, lost reply, client crash (orphans). **Exactly-once** is not achievable by retry alone.
- **Reliable multicast:** ACK implosion, NACK + suppression (SRM), hierarchical feedback. **Atomic multicast** = all or none. **Virtual synchrony** = views + flush.
- **2PC:** vote then decide, 4N messages, **blocks** if the coordinator fails when participants are READY. **3PC:** adds PRECOMMIT, 6N messages, non-blocking but assumes no partitions.
- **Recovery rules:** participant with no `<ready>` aborts. `<ready>` with no decision asks others. Coordinator with `<begin-commit>` but no decision aborts.

