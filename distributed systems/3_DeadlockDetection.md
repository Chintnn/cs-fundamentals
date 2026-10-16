# Distributed Systems: Unit 3 Notes (Distributed Deadlock Detection)

---

## 1. Deadlock Fundamentals

### 1.1 Definition
A **deadlock** is a state where a set of processes are each **waiting for a resource held by another process in the set**, so none can ever proceed.

### 1.2 Four Necessary (Coffman) Conditions
All four must hold together:

| Condition | Meaning |
|---|---|
| **Mutual exclusion** | A resource can be used by only one process at a time |
| **Hold and wait** | A process holds some resources while waiting for others |
| **No preemption** | Resources cannot be forcibly taken away |
| **Circular wait** | A cycle of processes exists, each waiting for the next |

### 1.3 Types of Deadlock in Distributed Systems
- **Resource deadlock:** waiting for locks or resources such as files or database rows. This is the main focus of this unit.
- **Communication deadlock:** processes wait for messages from each other.

### 1.4 Request Models
The model decides what counts as a deadlock.

| Model | Process is unblocked when | Deadlock condition |
|---|---|---|
| **Single-resource** | The one resource it waits for is granted | A cycle |
| **AND model** | **All** requested resources are granted | A cycle |
| **OR model** | **Any one** requested resource is granted | A **knot** (a cycle is not enough) |
| **P-out-of-Q** | Any P of Q requests are granted | Generalization of AND and OR |

### 1.5 Wait-For Graph (WFG)
- **Nodes** are processes or transactions.
- A **directed edge Pi → Pj** means Pi is waiting for a resource held by Pj.
- In the AND model, a **cycle means deadlock**.
- A **local WFG** is kept at each site. The **global WFG** is the union of all local graphs.

```
 Site S1        Site S2        Site S3
  P1 ---> P2     P2 ---> P3     P3 ---> P1

 Each local WFG has NO cycle, but the global WFG does:
     P1 --> P2 --> P3 --> P1   => DEADLOCK
```

**Simulation output:**
```
S1 local WFG: {'P1': ['P2']} -> no cycle
S2 local WFG: {'P2': ['P3']} -> no cycle
S3 local WFG: {'P3': ['P1']} -> no cycle
GLOBAL WFG: ... -> P1 -> P2 -> P3 -> P1
=> no local cycle anywhere, but a global deadlock exists
```
This is why a distributed system cannot rely on each site checking its own graph.

### 1.5.1 Why It Is Harder Than in a Centralized System
- There is no shared memory and **no global state**.
- The WFG is spread across sites and must be assembled using messages, which are delayed.
- By the time a graph is assembled it may be **stale**, so false deadlocks can be detected.

---

## 2. Deadlock Handling Strategies

```
 Deadlock handling
 ├── Prevention  (make one Coffman condition impossible)
 ├── Avoidance   (grant only if the state stays safe)
 └── Detection and Resolution   (allow, detect, then recover)
```

### 2.1 Deadlock Prevention
Break one of the four conditions in advance.

| Approach | Breaks | Drawback |
|---|---|---|
| Request **all resources up front** (atomically) | Hold and wait | Low utilization, resources are hard to predict |
| **Resource ordering** (always request in increasing order) | Circular wait | Hard to impose globally across sites |
| **Preemption** (take resources back) | No preemption | Rollback cost |

**Timestamp-based prevention (used in distributed databases).** Each transaction gets a unique timestamp. A smaller timestamp means an older transaction.

| Scheme | If Ti (requester) requests a resource held by Tj |
|---|---|
| **Wait-Die** (non-preemptive) | If Ti is **older**, Ti **waits**. If Ti is **younger**, Ti **dies** (aborts, restarts with the **same timestamp**). |
| **Wound-Wait** (preemptive) | If Ti is **older**, Ti **wounds** Tj (Tj aborts). If Ti is **younger**, Ti **waits**. |

- In both schemes waits only go one way (old to young in Wait-Die, young to old in Wound-Wait), so **no cycle can form**.
- Keeping the original timestamp on restart gives eventual seniority, so **no starvation**.
- A downside is that transactions may be aborted even when no deadlock would have occurred.

### 2.2 Deadlock Avoidance
Before each grant, check whether the resulting state is **safe** (Banker's algorithm).
- It needs a **global view** of all allocations and future maximum demands.
- In a distributed system that is expensive, and the state is stale by the time it is collected.
- Future needs are rarely known, so avoidance is **impractical** here.

### 2.3 Deadlock Detection and Resolution
This is the **practical choice**. Let deadlocks happen, detect them with a WFG, then break them by aborting or rolling back a victim. The rest of this unit covers it.

### 2.4 Timeouts (the lazy method)
Abort a transaction that waits longer than a limit. It is simple but gives false positives (slow is not deadlocked), and the right timeout value is hard to choose.

### 2.5 Ostrich Approach
Ignore the problem and let users restart. Used when deadlocks are rare.

---

## 3. Issues in Deadlock Detection and Resolution

### 3.1 Detection Issues
Every correct detection algorithm must satisfy two properties:
1. **Progress (no undetected deadlocks):** every real deadlock is detected in finite time.
2. **Safety (no false deadlocks):** a reported deadlock must really exist.

**The phantom deadlock problem.** Because sites report asynchronously, the assembled graph may combine old and new edges that never existed together.

**Phantom deadlock example (centralized detection):**
1. S1 reports the edge **P1 → P2** (P1 waits for a resource held by P2).
2. P2 then releases that resource, so P1 acquires it and the edge disappears.
3. P3 now requests a resource held by P1 (**P3 → P1**), and P2 requests one held by P3 (**P2 → P3**). These edges are reported.
4. The coordinator combines the old P1→P2 with the new edges and sees a cycle that **never existed**.

**Simulation output:**
```
Coordinator's merged graph: {...} -> P1 -> P2 -> P3 -> P1
But edge P1->P2 no longer exists (P2 released R_a)!
REAL graph: {'P2': ['P3'], 'P3': ['P1']} -> no cycle
=> PHANTOM (false) deadlock detected; a victim would be aborted for nothing
```

Other detection issues:
- **Maintaining the WFG:** where it is stored and how it is updated
- **Deciding when to search:** on every wait (costly) vs periodically vs after a timeout
- **Communication cost** and **detection delay**
- **Failure of sites** or lost messages during detection
- **Multiple initiators:** several sites may detect the same deadlock simultaneously and abort different victims, so the resolution is duplicated

### 3.2 Resolution Issues
Once a cycle is found, it must be broken by **aborting or rolling back** at least one process.

| Issue | Description |
|---|---|
| **Victim selection** | Choose the process with the least cost: lowest priority, least work done, fewest resources held, fewest rollbacks, youngest |
| **Starvation** | The same process must not be chosen every time, so track the number of times it was a victim and raise its priority |
| **Minimality** | Abort as few processes as possible |
| **Rollback cost** | Roll back only as far as needed (to a checkpoint), not to the start |
| **Consistent cleanup** | Release the victim's resources at **all** sites and remove its edges from every WFG, otherwise stale edges cause phantoms |
| **Single victim** | Ensure only **one** process is aborted per cycle, even if several detectors fire |

---

## 4. Classification of Detection Algorithms

```
 Deadlock detection algorithms
 ├── Centralized        (one control site holds the global WFG)
 ├── Distributed
 │     ├── Path-pushing          (send whole paths of the WFG)
 │     ├── Edge-chasing          (send small probe messages)
 │     ├── Diffusion computation (query/reply, used for OR model)
 │     └── Global state detection (snapshot-based)
 └── Hierarchical       (sites in a tree of controllers)
```

---

## 5. Centralized Deadlock Detection

### 5.1 Idea
One site, the **coordinator (control site)**, builds and maintains the **global WFG** and searches it for cycles.

```
        [Coordinator: global WFG]
         ^        ^        ^
         |        |        |   (edge updates)
        S1       S2       S3
```

### 5.2 When Do Sites Send Information?
1. **Continuously:** every time an edge is added or removed, the site tells the coordinator.
2. **Periodically:** sites send batches of changes at intervals.
3. **On request:** the coordinator asks sites for their graph when it wants to check.

### 5.3 Evaluation

| Pros | Cons |
|---|---|
| Simple to design and implement | **Single point of failure** |
| Easy to pick one victim | Bottleneck and heavy message load near the coordinator |
| Complete view of the graph | **Phantom deadlocks** from message delays |
| | Poor scalability |

Phantoms can be reduced if messages arrive in order, or by using **logical timestamps** or **verification** before aborting. Ho-Ramamurthy's algorithms address this directly.

---

## 6. Ho-Ramamurthy Algorithm

Ho and Ramamurthy proposed two **centralized** algorithms that **avoid phantom deadlocks** by checking that the reported edges are **consistent**. One site acts as the **control site**.

### 6.1 Two-Phase Algorithm

**Setup:** each site keeps a **status table** showing, for each of its transactions, which resources it holds and waits for.

**Steps:**
1. **Phase 1:** the control site requests the status table from every site, builds a WFG, and looks for a cycle.
2. If a cycle is found, **Phase 2** starts: the control site requests the status tables again.
3. It builds a second WFG using only the edges that appear **unchanged in both reports**.
4. A deadlock is declared only if the **cycle still exists** in this filtered graph.

**Simulation output:**
```
Case A (phantom): report1 = P1->P2, P2->P3, P3->P1
                  report2 = P2->P3, P3->P1          (P1->P2 vanished)
  cycle in both reports -> no cycle        (phantom rejected)
Case B (real):    report1 = P1->P2, P2->P3, P3->P1
                  report2 = same edges
  cycle in both reports -> P1 -> P2 -> P3 -> P1   (real deadlock)
```

Edges that existed in the first report but vanished by the second were transient, so any cycle depending on them is discarded.

- **Pros:** removes phantom deadlocks
- **Cons:** two rounds of messages for every check, so higher delay and overhead

### 6.2 One-Phase Algorithm

**Setup:** each site maintains **two** tables:
- **Process (transaction) status table:** the resources each local process holds or waits for
- **Resource status table:** the transactions that hold or wait for each local resource

**Steps:**
1. The control site requests the tables from all sites in **one round**.
2. It builds the WFG using **only the transactions whose entries are consistent** across the process table and the resource table. Inconsistent entries are ignored.
3. A cycle in this graph means a **real deadlock**.

### 6.3 Comparison

| Feature | Two-phase | One-phase |
|---|---|---|
| Rounds of messages | 2 | **1** |
| Storage per site | One table | **Two tables** (more) |
| Detection delay | Higher | Lower |
| How phantoms are avoided | Compare two successive reports | Compare process table with resource table |
| Control site | Needed | Needed |

Both share the weaknesses of centralized schemes: a single control site that can fail and become a bottleneck.

---

## 7. Distributed Deadlock Detection Algorithms

**Idea:** no single site holds the whole graph. All sites **cooperate**, and the responsibility of detection is shared.

### 7.1 Path-Pushing (WFG-based)
Each site builds a local WFG and **pushes** its paths (or the whole local graph) to other sites. Sites merge the received paths with their own and check for cycles.

**Obermarck's algorithm (System R\*):**
- Each site keeps a local WFG with a special external node **Ex** representing remote transactions.
- Sites send **potential-deadlock paths** like `Ex → Ti → Tj → Ex`.
- To cut messages, a path is sent **only if the timestamp ordering allows it** (for example only when Ti's ID is greater than Tj's, so each cycle is reported by exactly one site).
- **Drawback:** many large messages, and **phantom** deadlocks are possible.

### 7.2 Edge-Chasing (Probe-Based): Chandy-Misra-Haas (AND Model)

**Idea:** instead of sending graphs, send tiny **probe messages** along the WFG edges. If a probe **returns to its initiator**, there is a cycle.

**Probe format:** `probe(i, j, k)` means a probe **initiated by Pi**, **sent by Pj** to **Pk**.

**Algorithm:**
1. When Pi becomes blocked and suspects a deadlock, it sends `probe(i, i, k)` to every process Pk it waits for.
2. When a blocked process Pk receives `probe(i, j, k)`:
   - If Pk has **not already forwarded** a probe for initiator i (a flag `dependent_k[i]` is false), it sets the flag and **forwards** `probe(i, k, m)` to every Pm it waits for.
   - If **k = i** (the probe came back to the initiator), Pi declares **DEADLOCK**.
3. An **active (non-blocked)** process discards the probe.

**Simulation** (P1 → P2 → P3 → {P4, P5}, P4 → P1, across sites S1, S2, S3):
```
P1 is blocked -> sends probes (P1,P1,k)
  probe(P1,P1,P2)  local       P1@S1 -> P2@S1
  probe(P1,P2,P3)  inter-site  P2@S1 -> P3@S2
  probe(P1,P3,P4)  inter-site  P3@S2 -> P4@S3
  probe(P1,P3,P5)  inter-site  P3@S2 -> P5@S3
  probe(P1,P4,P1)  inter-site  P4@S3 -> P1@S1
*** P1 received its own probe => DEADLOCK (cycle through P1) ***

--- initiator P5 (waits for nobody) ---
probe died out => no deadlock
```

```
 P1 --probe--> P2 --probe--> P3 --probe--> P4
 ^                            \              |
 |                             --probe--> P5 (active, drops it)
 +----------------- probe returns ------------+
```

**Why it is good:**
- Probes are **small and fixed size**: three numbers.
- **No graph is constructed** anywhere.
- It detects **no phantom deadlocks** because a probe only travels along edges that exist while the process is blocked, and a process that becomes active discards it.
- **Cost:** at most **m(n−1)/2** messages for a deadlock cycle involving **m** processes at **n** sites, and detection delay of **O(n)**.

### 7.3 Diffusion Computation (OR Model)
For the OR model, a cycle is not enough, so a **knot** must be detected.
- The initiator sends **query** messages to all processes it waits on, and these **diffuse** outward.
- A blocked process forwards the query on first receipt and counts replies. It sends a **reply** back to the sender only after **all** its own queries are answered.
- If the initiator gets replies to all its queries, **every** process it depends on is blocked, so there is a deadlock.
- Example: **Chandy-Misra-Haas OR-model algorithm**.

### 7.4 Global State Detection
Take a **consistent global snapshot** (as in Chandy-Lamport from Unit 2) and look for a deadlock in the snapshot. Since a deadlock is a **stable property** (once true, it stays true), a deadlock found in a consistent snapshot is a real deadlock. Example: **Bracha-Toueg** algorithm.

### 7.5 Comparison of Distributed Approaches

| Feature | Path-pushing | Edge-chasing | Diffusion | Global state |
|---|---|---|---|---|
| Message content | Paths or graphs | Small probes | Query/reply | Snapshot data |
| Message size | Large | **Small** | Small | Large |
| Model | AND | AND | **OR** | AND/OR |
| Phantom risk | Yes | **No** | No | No |
| Example | Obermarck | Chandy-Misra-Haas | CMH (OR) | Bracha-Toueg |

**General evaluation of distributed algorithms:**
- No single point of failure and no central bottleneck
- Harder to design and prove correct
- Several sites may detect the same deadlock, so resolution needs coordination

---

## 8. Hierarchical Deadlock Detection

### 8.1 Idea
Organize the sites as a **tree of controllers**. Each controller is responsible for detecting deadlocks among the sites **under it**.

```
              Root controller
              /             \
           C1                 C2
          /  \               /  \
        S1    S2           S3    S4
```

- **Leaf controllers** manage the local WFG of their site.
- A **non-leaf controller** keeps the **union of the WFGs of its children** (only the part relevant for inter-child dependencies).
- A deadlock is detected by the **lowest controller whose subtree contains all the sites involved** in the cycle (the lowest common ancestor).

### 8.2 Why This Works
- A deadlock **inside one cluster** is found by that cluster's controller, so **the root is never involved**.
- Only deadlocks that **span clusters** are pushed up the tree.

### 8.3 Example

**Simulation output:**
```
Tree: Root{C1{S1,S2}, C2{S3,S4}}
Case 1: cycle uses edges at S3,S4 -> detected at C2 (Root never involved)
Rule: a deadlock is detected by the lowest controller whose subtree
      covers all sites in the cycle
```

Corrected examples for the notes:
- **Cycle between S1 and S2 only:** detected at **C1**.
- **Cycle between S3 and S4 only:** detected at **C2**.
- **Cycle P1→P2 (S1), P2→P3 (S2), P3→P4 (S3), P4→P1 (S4):** touches both subtrees, so it is detected at the **Root**.

### 8.4 Hierarchical Algorithms

| Algorithm | Notes |
|---|---|
| **Menasce-Muntz** | Controllers form a tree. Each leaf reports its local resource and wait edges **upward**, and each controller merges its children's graphs and looks for cycles. Phantoms are possible if reports are asynchronous. |
| **Ho-Ramamurthy (hierarchical)** | Sites are grouped into **clusters**, each with its own controller, arranged in a tree. Each controller runs the **one-phase (or two-phase) idea** over its cluster, and results are escalated upward. |

### 8.5 Evaluation

| Pros | Cons |
|---|---|
| Less load on any single site than the centralized scheme | Tree construction and maintenance is complex |
| **Local deadlocks stay local**, which is cheap | The root can become a bottleneck for global deadlocks |
| Fewer long-distance messages, better scalability | A failed controller cuts off its subtree |
| Failure of one controller affects only its cluster | Cross-cluster detection has higher delay |

---

## 9. Master Comparison

| Algorithm | Type | Messages | Phantoms | Single point of failure |
|---|---|---|---|---|
| Completely centralized | Centralized | O(edges) per update | **Yes** | **Yes** |
| Ho-Ramamurthy two-phase | Centralized | 2 rounds of status tables | **No** | Yes |
| Ho-Ramamurthy one-phase | Centralized | 1 round, 2 tables | **No** | Yes |
| Obermarck | Path-pushing | Up to n(n−1)/2 | Possible | No |
| Chandy-Misra-Haas | Edge-chasing | ≤ m(n−1)/2 probes | **No** | No |
| Menasce-Muntz | Hierarchical | Tree-structured | Possible | Per cluster |

---

## 10. Likely Interview Questions

1. **What is a deadlock, and what are the necessary conditions?**
   Processes waiting in a cycle for resources held by each other. Mutual exclusion, hold and wait, no preemption, circular wait.

2. **How is deadlock in a distributed system different from a centralized one?**
   There is no global state, the WFG is spread over sites, and message delay makes a consistent picture hard, so phantom deadlocks can occur.

3. **What is a wait-for graph?**
   A directed graph where an edge Pi → Pj means Pi is waiting for Pj. A cycle means deadlock in the AND model.

4. **What is a phantom deadlock? How can it be avoided?**
   A cycle detected from inconsistent or stale information that never existed. Avoid it with the Ho-Ramamurthy two-phase or one-phase checks, edge-chasing probes, or consistent snapshots.

5. **Prevention vs avoidance vs detection?**
   Prevention breaks a Coffman condition in advance. Avoidance needs safe-state checks, which is impractical. Detection lets deadlocks occur and then resolves them, which is the practical choice.

6. **Wait-Die vs Wound-Wait?**
   Wait-Die: older waits, younger dies. Wound-Wait: older wounds (aborts) the holder, younger waits. Both are timestamp-based and starvation-free.

7. **Explain centralized deadlock detection and its drawbacks.**
   A control site keeps the global WFG. Drawbacks are a single point of failure, a bottleneck, and phantom deadlocks.

8. **Explain the Ho-Ramamurthy algorithm.**
   A centralized detector that avoids phantoms. The two-phase version compares two successive reports and keeps only unchanged edges. The one-phase version uses a process table and a resource table and keeps only consistent entries.

9. **How does the Chandy-Misra-Haas algorithm work?**
   A blocked process sends probe(i, j, k) along wait edges. A blocked process forwards the probe once. If the probe returns to its initiator, there is a deadlock.

10. **What is the message complexity of Chandy-Misra-Haas?**
    At most m(n−1)/2 messages for m processes at n sites, with O(n) delay.

11. **What is the difference between the AND model and the OR model?**
    AND: need all requests granted, deadlock is a cycle. OR: need any one granted, deadlock is a knot.

12. **How does hierarchical detection work?**
    Controllers form a tree, each merges its children's graphs, and a deadlock is found at the lowest common ancestor controller of the sites involved.

13. **How do you choose a deadlock victim?**
    Pick the lowest-cost process (least work, fewest resources, lowest priority), and track abort counts to avoid starvation.

14. **Why is deadlock avoidance not used in distributed systems?**
    It needs global state and knowledge of future resource demands, both unavailable.

---

## 11. Quick Revision Sheet

- **Coffman conditions:** mutual exclusion, hold and wait, no preemption, circular wait.
- **WFG:** edge Pi → Pj = Pi waits for Pj. **AND model: cycle. OR model: knot.**
- **Strategies:** prevention, avoidance (impractical), **detection plus resolution (practical)**.
- **Wait-Die:** old waits, young dies. **Wound-Wait:** old wounds, young waits. Keep the original timestamp.
- **Detection correctness:** progress (no missed deadlock) and safety (no false deadlock).
- **Phantom deadlock:** a cycle built from stale or inconsistent edges.
- **Resolution issues:** victim selection, starvation, minimal aborts, rollback cost, cleaning up all sites.
- **Centralized:** one control site, simple, but single point of failure plus phantoms.
- **Ho-Ramamurthy:** **two-phase** = two reports, keep unchanged edges. **One-phase** = process table plus resource table, keep consistent entries. Both are centralized and phantom-free.
- **Distributed:** path-pushing (Obermarck), **edge-chasing (Chandy-Misra-Haas, probe(i,j,k), m(n−1)/2 messages)**, diffusion (OR model), global state (Bracha-Toueg).
- **Hierarchical:** controller tree, detection at the **lowest common ancestor**, local deadlocks stay local (Menasce-Muntz, Ho-Ramamurthy hierarchical).

