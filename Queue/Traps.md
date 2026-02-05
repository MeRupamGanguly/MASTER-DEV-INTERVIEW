# Kafka vs RabbitMQ – Interview Traps with Explanations 

---

To truly master these traps, you need to understand the **underlying philosophy** of each system. Interviewers use these questions to see if you understand *distributed systems theory* or if you’ve just memorized a feature list.

Below is a deep-dive breakdown of the traps, grouped by the technical "themes" that interviewers care about most.

---

## 1. The Architectural Foundations

**Traps: 1, 7, 9, 21, 27, 32, 40**

The fundamental difference is that **RabbitMQ is a "Smart Broker/Dumb Consumer" The broker keeps track of who got what and actively pushes messages to consumers.** model, while **Kafka is a "Dumb Broker/Smart Consumer The broker stores an append‑only log; consumers pull messages and manage their own progress."** model.

* **Push vs. Pull (Trap 7):** RabbitMQ pushes messages to you. This is great for low latency but dangerous if the consumer is slow (hence the need for "Prefetch" limits). Kafka makes the consumer pull. This shifts the "burden of work" to the consumer, making the broker incredibly fast.
* **The Log vs. The Queue (Trap 1, 9):** RabbitMQ treats messages like a post office (once delivered, the letter is gone). Kafka treats messages like a recorded tape (the data stays there, and you can rewind the tape).
* **Zero-Copy (Trap 21):** Kafka is faster for throughput because it uses a Linux kernel trick called `sendfile()`. It moves data from the disk directly to the network card, skipping the "middleman" of the application's memory.

---

## 2. The "Reliability" Spectrum

Reliability is about how many copies exist, who confirms writes, and how failures are detected.

**Traps: 3, 4, 11, 24, 26, 31, 34, 35**

These traps test if you know how to prevent data loss.

* **ACK Levels (Trap 35):** This is the most common trap. `acks=all` in Kafka ensures the data is written to multiple servers before the producer gets a "success" message. In RabbitMQ, you use "Publisher Confirms." Replication factor controls how many copies of data exist. More replicas → safer but slower. acks in Kafka: acks=0, acks=1, acks=all. acks=all is safest because all in‑sync replicas must confirm.
* **The ISR (In-Sync Replicas) (Trap 34):** If a Kafka broker falls behind the leader, it is kicked out of the ISR. This prevents a "slow" server from dragging down the whole cluster's performance.
* **Split Brain (Trap 31):** In a network failure, two servers might both think they are the "leader." Both systems solve this with **Quorum** (majority rule). If you have 3 nodes, you need 2 to agree.

---

## 3. Ordering & Parallelism

Ordering is guaranteed only at certain granularity; parallelism comes from splitting work (partitions/queues).

Kafka: ordering guaranteed per partition. To keep related events ordered, send them to the same partition using a key.

RabbitMQ: ordering is per queue; complex routing can send messages to different queues.
**Traps: 2, 15, 22, 23, 30, 37**

Interviewer's favorite: "How do I scale while keeping things in order?"

* **Partitioning (Trap 2, 30):** In Kafka, **1 Partition = 1 Consumer.** If you have 10 partitions and 11 consumers, the 11th consumer will sit idle. This is a massive trap for beginners.
* **Changing Partitions (Trap 22, 37):** If you go from 10 to 20 partitions, your "Key Hashing" changes. A user’s data that used to go to Partition 1 might now go to Partition 5, **breaking the chronological order** of their events.
* **Priority (Trap 23):** RabbitMQ handles "VIP" messages easily. Kafka doesn't care—everything is a sequence. You have to build "Priority" manually in Kafka by using different topics.

---

## 4. Message Guarantees & "Poison Pills"

Guarantees (at‑most‑once, at‑least‑once, exactly‑once) are tradeoffs between simplicity, performance, and complexity.

At‑least‑once: message may be delivered more than once (safe but may duplicate).

At‑most‑once: message delivered zero or one time (fast but may lose messages).

Exactly‑once: hard to achieve end‑to‑end; Kafka supports it internally with idempotent producers and transactions, but external systems must be idempotent too.

**Traps: 5, 14, 18, 25, 33, 39**

* **Exactly-Once (Trap 5, 25):** This is a "marketing" trap. Kafka supports exactly-once *internally*. But if your consumer writes to a database, and the database write succeeds but the Kafka commit fails, you still get a duplicate. You need **Idempotency** (checking if the ID exists) at the final destination.
* **DLQs (Trap 14, 33, 39):** A "Poison Pill" is a message that causes your code to crash every time it tries to process it. RabbitMQ handles this natively. In Kafka, you have to write code to "catch" the error and manually send that message to a "Side Topic" (your DIY Dead Letter Queue).

---

## 5. Performance, Latency, and Scalability

Throughput vs latency: Kafka is optimized for throughput (large volumes, batching); RabbitMQ is optimized for low latency per message.

**Traps: 6, 8, 10, 12, 13, 19, 28, 29, 36, 38**

* **Backpressure (Trap 10, 36):** If your consumers are slow, RabbitMQ's memory fills up, and it will eventually "pressure" the producer to stop. Kafka doesn't care; it just keeps writing to disk. The disk is the buffer.
* **Consumer Lag (Trap 29):** This is the #1 metric to monitor. If your "End Offset" is 1,000,000 and your "Current Offset" is 800,000, you are 200,000 messages behind.
* **TTL & Compaction (Trap 8, 38):** RabbitMQ deletes messages by time. Kafka usually does too, but it also has **Log Compaction**, which keeps only the *latest* version of a key (great for storing a user's current address).

---

## 6. Security & Ops

**Traps: 16, 17, 20, 27**

* **Erlang (Trap 27):** RabbitMQ is written in Erlang, which was built by Ericsson for phone switches. It's designed to never crash. Kafka is Java/Scala, which is more common but requires heavy JVM tuning (Garbage Collection).
* **The "When to Use" (Trap 20):** * **Kafka:** "I need to analyze 1 billion clicks to see user trends." (Big Data/Streaming)
* **RabbitMQ:** "I need to send an email, a push notification, and a receipt when a user clicks 'Buy'." (Task Routing/Microservices)



---

### Summary Table for Quick Recall

| If the Interviewer asks about... | Think of this Keyword |
| --- | --- |
| **Throughput / Scale** | Kafka (Partitions & Zero-copy) |
| **Complex Routing** | RabbitMQ (Exchanges & Bindings) |
| **Data Retention** | Kafka (Immutable Log) |
| **Low Latency** | RabbitMQ (Push model) |
| **Retrying Errors** | RabbitMQ (Native DLQ) |

___

## Trap 1: Kafka vs RabbitMQ – Messaging Model
- **Trap:** "Are Kafka and RabbitMQ the same?"  
- **Answer:** No. Kafka is a distributed log streaming platform; RabbitMQ is a message broker.  
- **Explanation:** Kafka persists an append-only log and lets consumers pull offsets; RabbitMQ routes messages and the broker tracks delivery/acks.  
- **Follow-up Q&A:**  
  - **Q:** Why is Kafka called a log?  
  - **A:** Because it stores immutable records in sequence so consumers can replay history.

---

## Trap 2: Message Ordering
- **Trap:** "Does Kafka guarantee ordering?"  
- **Answer:** Only within a partition.  
- **Explanation:** Ordering across partitions is not guaranteed; to preserve order for related keys, use a partitioning key.  
- **Follow-up Q&A:**  
  - **Q:** How to get global ordering?  
  - **A:** Use a single partition (sacrifices throughput) or design a custom partitioner/migration.

---

## Trap 3: Durability
- **Trap:** "Are Kafka messages always durable?"  
- **Answer:** Not automatically—depends on replication and acks.  
- **Explanation:** `replication.factor` and `acks` determine durability; leader-only writes risk loss.  
- **Follow-up Q&A:**  
  - **Q:** Best settings for durability?  
  - **A:** `replication.factor >= 3` and `acks=all`.

---

## Trap 4: Consumer Acknowledgment
- **Trap:** "How do consumers acknowledge messages?"  
- **Answer:** Kafka: offsets (consumer-managed); RabbitMQ: explicit ACK/NACK to broker.  
- **Explanation:** Kafka commits offsets to track progress; RabbitMQ removes messages on ACK.  
- **Follow-up Q&A:**  
  - **Q:** What if Kafka consumer crashes before commit?  
  - **A:** Messages may be reprocessed (at-least-once).

---

## Trap 5: Exactly-Once Semantics
- **Trap:** "Does Kafka guarantee exactly-once?"  
- **Answer:** Kafka supports exactly-once semantics (EOS) with idempotent producers + transactions; not automatic.  
- **Explanation:** EOS prevents duplicates inside Kafka; external sinks still need idempotency.  
- **Follow-up Q&A:**  
  - **Q:** What components enable EOS?  
  - **A:** Idempotent producer, transactional producer, and transactional consumer handling.

---

## Trap 6: Scalability
- **Trap:** "Which scales better, Kafka or RabbitMQ?"  
- **Answer:** Kafka scales more naturally horizontally via partitions; RabbitMQ requires more operational sharding.  
- **Explanation:** Kafka partitions distribute load across brokers; RabbitMQ queues are node-bound unless sharded.  
- **Follow-up Q&A:**  
  - **Q:** How to scale RabbitMQ?  
  - **A:** Use clustering, federation, or sharding plugins; design queues to avoid single-node hotspots.

---

## Trap 7: Push vs Pull
- **Trap:** "Does Kafka push messages to consumers?"  
- **Answer:** No—Kafka is pull-based; RabbitMQ is push-based (with consumer prefetch controls).  
- **Explanation:** Pull gives consumers control over consumption rate and backpressure.  
- **Follow-up Q&A:**  
  - **Q:** How does RabbitMQ avoid overwhelming consumers?  
  - **A:** Use QoS/prefetch settings to limit in-flight messages.

---

## Trap 8: Message TTL
- **Trap:** "Can Kafka set per-message TTL?"  
- **Answer:** No—Kafka uses topic-level retention (time/size), not per-message TTL.  
- **Explanation:** RabbitMQ supports per-message and per-queue TTLs; Kafka retention is coarse-grained.  
- **Follow-up Q&A:**  
  - **Q:** How to emulate TTL in Kafka?  
  - **A:** Use compacted topics or a consumer-side filter and separate retention policies.

---

## Trap 9: Replay
- **Trap:** "Can RabbitMQ replay consumed messages like Kafka?"  
- **Answer:** Historically no; RabbitMQ Streams (3.9+) adds replay capability. Kafka supports replay via offsets.  
- **Explanation:** Kafka’s log model inherently supports replay; RabbitMQ classic queues do not.  
- **Follow-up Q&A:**  
  - **Q:** How to replay in Kafka?  
  - **A:** Reset consumer group offsets or use a new consumer group.

---

## Trap 10: Backpressure
- **Trap:** "How do Kafka and RabbitMQ handle slow consumers?"  
- **Answer:** Kafka: consumers pull at their pace; RabbitMQ: queues can grow and may block producers or require flow control.  
- **Explanation:** Kafka’s model decouples producers from slow consumers; RabbitMQ needs careful flow control.  
- **Follow-up Q&A:**  
  - **Q:** What happens when RabbitMQ queue grows too large?  
  - **A:** Memory pressure, disk paging, or producer blocking depending on broker config.

---

## Trap 11: High Availability
- **Trap:** "Does Kafka guarantee high availability out of the box?"  
- **Answer:** Only if configured with replication and proper ISR settings.  
- **Explanation:** Leader + followers + ISR provide HA; misconfiguration can cause data loss.  
- **Follow-up Q&A:**  
  - **Q:** What is ISR?  
  - **A:** In-Sync Replicas—followers that have caught up with the leader.

---

## Trap 12: Latency
- **Trap:** "Which has lower latency?"  
- **Answer:** RabbitMQ typically has lower per-message latency; Kafka favors throughput and batching.  
- **Explanation:** Kafka’s disk-based, batched writes increase throughput but can add latency.  
- **Follow-up Q&A:**  
  - **Q:** How to reduce Kafka latency?  
  - **A:** Tune batch sizes, linger.ms, and use faster disks.

---

## Trap 13: Large Messages
- **Trap:** "Can Kafka handle very large messages?"  
- **Answer:** Not ideal—large messages (>1MB) are discouraged; both systems perform poorly with huge payloads.  
- **Explanation:** Large messages increase GC, network, and disk overhead.  
- **Follow-up Q&A:**  
  - **Q:** Best practice for large payloads?  
  - **A:** Store payload in object storage (S3) and send a reference in the message.

---

## Trap 14: Dead Letter Queue
- **Trap:** "Does Kafka have a built-in DLQ?"  
- **Answer:** No native DLQ; implement by producing failed messages to a dedicated topic. RabbitMQ has DLX/DLQ support.  
- **Explanation:** Kafka requires application-level handling for poison messages.  
- **Follow-up Q&A:**  
  - **Q:** How to design a Kafka DLQ?  
  - **A:** Create a failure topic with metadata (error reason, original offset) and monitor it.

---

## Trap 15: Ordering Across Consumers
- **Trap:** "Can multiple consumers maintain strict order?"  
- **Answer:** Only if they consume from the same partition and one consumer per partition is used.  
- **Explanation:** Kafka enforces ordering per partition; multiple consumers in a group split partitions.  
- **Follow-up Q&A:**  
  - **Q:** What if you need ordered processing and high throughput?  
  - **A:** Partition by key and ensure related events go to same partition; scale by increasing partitions for other keys.

---

## Trap 16: Security
- **Trap:** "Is Kafka secure by default?"  
- **Answer:** No—security (TLS, SASL, ACLs) must be explicitly configured. RabbitMQ supports TLS and pluggable auth backends.  
- **Explanation:** Default Kafka is often plaintext; production requires encryption and auth.  
- **Follow-up Q&A:**  
  - **Q:** How to protect Kafka data at rest?  
  - **A:** Use OS/disk encryption (LUKS, cloud provider encryption) or application-level encryption.

---

## Trap 17: Monitoring
- **Trap:** "How do you monitor Kafka vs RabbitMQ?"  
- **Answer:** Kafka: JMX metrics, consumer lag tools (Burrow, Cruise Control); RabbitMQ: management UI, Prometheus metrics.  
- **Explanation:** Key metrics: consumer lag, broker health, under-replicated partitions, queue lengths.  
- **Follow-up Q&A:**  
  - **Q:** How to measure consumer lag?  
  - **A:** Compare latest partition offset to consumer committed offset.

---

## Trap 18: Transactions
- **Trap:** "Does RabbitMQ support transactions like Kafka?"  
- **Answer:** RabbitMQ supports AMQP transactions but they are heavy; Kafka supports producer transactions for atomic writes.  
- **Explanation:** Kafka transactions are designed for high-throughput exactly-once flows; RabbitMQ transactions are less common in practice.  
- **Follow-up Q&A:**  
  - **Q:** When to use Kafka transactions?  
  - **A:** When you need atomic writes across multiple partitions/topics.

---

## Trap 19: Partition Rebalancing
- **Trap:** "What happens when a new Kafka consumer joins a group?"  
- **Answer:** Kafka triggers a rebalance that redistributes partitions among consumers.  
- **Explanation:** Rebalance pauses consumption briefly and can cause duplicate processing if offsets are not handled carefully.  
- **Follow-up Q&A:**  
  - **Q:** How to reduce rebalance impact?  
  - **A:** Use cooperative rebalancing, increase session timeouts, or use sticky partition assignment.

---

## Trap 20: Use Case
- **Trap:** "When should you use Kafka vs RabbitMQ?"  
- **Answer:** Kafka for event streaming, replay, analytics; RabbitMQ for complex routing, RPC, and task queues.  
- **Explanation:** Choose based on retention/replay needs vs routing/ack semantics.  
- **Follow-up Q&A:**  
  - **Q:** Example: e-commerce checkout vs stock ticker?  
  - **A:** Checkout tasks → RabbitMQ (task queue); stock ticker → Kafka (high-throughput stream).

---

## Trap 21: Which is faster?
- **Trap:** "Which is faster overall?"  
- **Answer:** Context matters—RabbitMQ often has lower latency per message; Kafka achieves much higher throughput.  
- **Explanation:** Kafka uses zero-copy and sequential disk I/O for throughput; RabbitMQ optimizes for low-latency delivery.  
- **Follow-up Q&A:**  
  - **Q:** What is zero-copy?  
  - **A:** OS-level optimization that moves data from disk to network without copying into user-space buffers.

---

## Trap 22: Kafka Ordering and Partition Changes
- **Trap:** "What happens to ordering if you increase partitions later?"  
- **Answer:** Increasing partitions can change partition assignment and break ordering for keys hashed differently.  
- **Explanation:** Partition count affects key-to-partition mapping; re-partitioning requires migration strategies.  
- **Follow-up Q&A:**  
  - **Q:** How to preserve ordering when increasing partitions?  
  - **A:** Use a custom partitioner or migrate keys carefully with a controlled rollout.

---

## Trap 23: Priority Queue in Kafka
- **Trap:** "Can you implement a priority queue in Kafka?"  
- **Answer:** Not natively. Simulate with separate topics (priority-high, priority-low) and consumer logic.  
- **Explanation:** Kafka is append-only and does not support per-message priority semantics.  
- **Follow-up Q&A:**  
  - **Q:** Downsides of simulating priority with topics?  
  - **A:** Increased complexity, potential duplication of consumers, and ordering challenges.

---

## Trap 24: RabbitMQ Message Lifecycle
- **Trap:** "What happens to a message after it's consumed in RabbitMQ?"  
- **Answer:** It is removed from the queue once the consumer ACKs it; if the consumer dies before ACK, the message is requeued.  
- **Explanation:** ACK semantics ensure at-least-once delivery unless auto-ack is used.  
- **Follow-up Q&A:**  
  - **Q:** How to ensure no duplicate processing?  
  - **A:** Implement idempotent consumers or use deduplication logic.

---

## Trap 25: Kafka Exactly Once Reality
- **Trap:** "Is Kafka's exactly-once truly exactly-once end-to-end?"  
- **Answer:** Within Kafka and transactional sinks yes; end-to-end requires idempotent handling in external systems.  
- **Explanation:** Kafka guarantees atomic writes inside the cluster; external side effects may still duplicate.  
- **Follow-up Q&A:**  
  - **Q:** How to achieve end-to-end exactly-once?  
  - **A:** Use idempotent writes in the sink or transactional connectors that support two-phase commit semantics.

---

## Trap 26: Zombie Leader
- **Trap:** "What is a Zombie Leader in Kafka?"  
- **Answer:** A broker that believes it is leader after network partition but is no longer the authoritative leader.  
- **Explanation:** Kafka uses epoch/fencing to reject stale leaders and protect data integrity.  
- **Follow-up Q&A:**  
  - **Q:** How does Kafka fence old leaders?  
  - **A:** By using leader epochs and rejecting requests from brokers with older epochs.

---

## Trap 27: Why RabbitMQ uses Erlang
- **Trap:** "Why is RabbitMQ implemented in Erlang?"  
- **Answer:** Erlang/BEAM is designed for massive concurrency, fault tolerance, and distributed systems.  
- **Explanation:** Lightweight processes and supervision trees make RabbitMQ resilient and able to handle many connections.  
- **Follow-up Q&A:**  
  - **Q:** What operational benefit does Erlang provide?  
  - **A:** Low memory per connection and robust process isolation for fault recovery.

---

## Trap 28: RabbitMQ Horizontal Scaling
- **Trap:** "Can RabbitMQ scale horizontally like Kafka?"  
- **Answer:** It can, but it’s more complex—requires clustering, sharding, or federation.  
- **Explanation:** Queues are typically node-local; scaling often needs architectural changes.  
- **Follow-up Q&A:**  
  - **Q:** What is federation in RabbitMQ?  
  - **A:** A way to connect brokers across datacenters to replicate messages between clusters.

---

## Trap 29: Consumer Lag
- **Trap:** "What is consumer lag and why does it matter?"  
- **Answer:** Lag = difference between latest partition offset and consumer committed offset; high lag indicates consumers can't keep up.  
- **Explanation:** Persistent lag can cause delayed processing and resource buildup.  
- **Follow-up Q&A:**  
  - **Q:** How to reduce lag?  
  - **A:** Scale consumers, optimize processing, increase partition parallelism.

---

## Trap 30: Partitions vs Consumers
- **Trap:** "If you have more consumers than partitions, what happens?"  
- **Answer:** Extra consumers remain idle; Kafka assigns at most one consumer per partition in a group.  
- **Explanation:** Partition count limits parallelism; plan partitions to match consumer scale.  
- **Follow-up Q&A:**  
  - **Q:** How to utilize extra consumers?  
  - **A:** Increase partitions or run multiple consumer groups for different workloads.

---

## Trap 31: Split Brain
- **Trap:** "What is split brain and how is it handled?"  
- **Answer:** Split brain occurs when cluster partitions and multiple nodes think they are leaders; resolved via quorum-based consensus.  
- **Explanation:** Kafka (KRaft/ZooKeeper) and RabbitMQ quorum queues use majority voting to avoid dual leaders.  
- **Follow-up Q&A:**  
  - **Q:** How to prevent split brain operationally?  
  - **A:** Use proper quorum sizes, stable networking, and fencing mechanisms.

---

## Trap 32: When NOT to Use Kafka
- **Trap:** "When would you avoid Kafka?"  
- **Answer:** For low-volume apps needing complex routing, simple RPC, or when ops resources are limited.  
- **Explanation:** Kafka adds operational complexity and is overkill for simple task queues.  
- **Follow-up Q&A:**  
  - **Q:** What’s a lightweight alternative?  
  - **A:** RabbitMQ or managed queue services (SQS, Pub/Sub) depending on needs.

---

## Trap 33: Dead Letter Exchange (RabbitMQ)
- **Trap:** "What is a Dead Letter Exchange (DLX)?"  
- **Answer:** An exchange where messages are routed when rejected, expired, or exceed delivery attempts.  
- **Explanation:** DLX + DLQ pattern isolates poison messages for inspection or retry.  
- **Follow-up Q&A:**  
  - **Q:** How to configure DLX?  
  - **A:** Set queue arguments (`x-dead-letter-exchange`, `x-dead-letter-routing-key`) on the original queue.

---

## Trap 34: Kafka High Availability Details
- **Trap:** "How does Kafka achieve HA?"  
- **Answer:** Through partition replication, leader election, and ISR tracking.  
- **Explanation:** Followers replicate leader segments; if leader fails, an in-sync follower is elected.  
- **Follow-up Q&A:**  
  - **Q:** What is under-replicated partition?  
  - **A:** A partition with fewer replicas in ISR than expected—indicator of risk.

---

## Trap 35: Kafka ACK Levels
- **Trap:** "What do ack=0, ack=1, ack=all mean?"  
- **Answer:** `acks=0`: no broker ack (fast, unsafe); `acks=1`: leader ack only; `acks=all`: all in-sync replicas must ack (safest).  
- **Explanation:** Higher ack levels increase durability at cost of latency.  
- **Follow-up Q&A:**  
  - **Q:** Which ack for production durability?  
  - **A:** `acks=all` with replication factor ≥ 3.

---

## Trap 36: Backpressure Revisited
- **Trap:** "What is backpressure and how is it handled?"  
- **Answer:** Backpressure is when consumers are slower than producers; Kafka’s pull model mitigates it, RabbitMQ uses flow control and prefetch.  
- **Explanation:** Proper consumer pacing and broker configs are essential to avoid overload.  
- **Follow-up Q&A:**  
  - **Q:** How to implement backpressure in producers?  
  - **A:** Monitor broker metrics and throttle producers or use client-side rate limiting.

---

## Trap 37: Changing Partitions
- **Trap:** "Can you change the number of partitions on the fly?"  
- **Answer:** You can increase partitions; you cannot decrease without recreating the topic.  
- **Explanation:** Increasing partitions changes key-to-partition mapping and can affect ordering.  
- **Follow-up Q&A:**  
  - **Q:** How to safely increase partitions?  
  - **A:** Plan partitioning strategy and migrate keys if ordering matters.

---

## Trap 38: Log Compaction
- **Trap:** "What is log compaction in Kafka?"  
- **Answer:** A retention mode that keeps only the latest value per key, useful for changelogs.  
- **Explanation:** Compaction preserves the latest state for keys while allowing retention of important updates.  
- **Follow-up Q&A:**  
  - **Q:** Use case for compaction?  
  - **A:** Kafka-backed key-value store or CDC (change data capture) topics.

---

## Trap 39: Poison Pill Messages
- **Trap:** "How do you handle poison pill messages that crash consumers?"  
- **Answer:** Catch exceptions, route failing messages to a DLQ, skip or quarantine the message, and alert.  
- **Explanation:** Repeated failures require isolating the message to avoid consumer churn.  
- **Follow-up Q&A:**  
  - **Q:** How to detect poison pills automatically?  
  - **A:** Track retry counts and move messages to DLQ after threshold.

---

## Trap 40: RabbitMQ Replay and Streams
- **Trap:** "Can RabbitMQ handle replay like Kafka?"  
- **Answer:** Classic RabbitMQ queues cannot replay; RabbitMQ Streams (3.9+) provide append-only, replayable streams similar to Kafka.  
- **Explanation:** Streams add long-term retention and replay semantics to RabbitMQ, narrowing the gap with Kafka.  
- **Follow-up Q&A:**  
  - **Q:** When to use RabbitMQ Streams vs Kafka?  
  - **A:** Use Streams if you need RabbitMQ routing features plus replay; use Kafka for very large-scale streaming ecosystems and tooling.

---
