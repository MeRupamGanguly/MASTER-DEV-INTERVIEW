# Redis Interview Traps

1 Is Redis only an in-memory cache
Answer: No. Redis is primarily in-memory but supports persistence via RDB and AOF.
Explanation: RDB snapshots provide fast point-in-time backups; AOF logs every write for higher durability. Use both for balanced durability and restart performance. Consider fsync settings and AOF rewrite behavior when designing durability.


2 What happens when Redis runs out of memory
Answer: Redis enforces maxmemory and applies the configured eviction policy.
Explanation: Choose noeviction to fail writes, or an eviction policy that matches workload semantics. Monitor memory usage and tune maxmemory and maxmemory-policy to avoid unexpected data loss.

3 Do expired keys get deleted immediately
Answer: Not necessarily. Redis uses lazy deletion and active expiration cycles.
Explanation: Expired keys may remain until accessed or until the background cleaner removes them. This design reduces CPU overhead but can leave stale keys in memory temporarily.

6 Is Redis multi-threaded
Answer: Redis is single-threaded for command execution; I/O threads exist in newer versions.
Explanation: The single-threaded model simplifies concurrency and reduces locking. Heavy workloads should be sharded or scaled horizontally.

7 Is Redis Pub/Sub reliable messaging
Answer: No. Pub/Sub is ephemeral and messages are lost if no subscriber is connected.
Explanation: For durable messaging, use Redis Streams with consumer groups and persistent storage.

11 Is Redis suitable for all workloads
Answer: No. Redis is ideal for low-latency, in-memory workloads but not for complex queries or very large persistent datasets.
Explanation: Use Redis for caching, session stores, leaderboards, counters, and ephemeral state. Use relational or analytical databases for joins, complex queries, and long-term storage.

13 Does Redis automatically shard data
Answer: Only in Cluster mode.
Explanation: Standalone Redis stores all keys on a single node. Cluster mode uses hash slots to distribute keys across nodes and requires careful key design to avoid cross-slot operations. Cross-slot operations (commands that touch multiple keys) can fail or require special handling (hash tags) in Cluster.

14 Are Redis hashes efficient for storing objects
Answer: Yes for many small fields; not for very large fields.
Explanation: Hashes use a compact encoding for small fields, saving memory compared to separate keys. When fields are large, the memory advantage disappears and you may be better off using separate keys or an external store.


## Ideal Candidate Responses for Interviews

- **Persistence question concise response:**  
  **“Redis is in-memory but supports RDB snapshots and AOF. RDB is fast for backups; AOF logs every write for durability. Many teams use both to balance restart speed and durability.”**

- **Eviction question concise response:**  
  **“Set `maxmemory` and choose a policy that matches your workload. For caches `allkeys-lru` is common; for session stores `volatile-*` policies can be safer. Monitor and test eviction behavior.”**

- **TTL question concise response:**  
  **“Expiration is lazy and probabilistic. Don’t assume immediate deletion; design for possible lingering keys and monitor memory.”**

- **Replication question concise response:**  
  **“Replication gives redundancy; use Sentinel or Cluster for automatic failover and sharding. Understand replica lag and split-brain risks.”**

- **Transactions question concise response:**  
  **“`MULTI/EXEC` is atomic but has no rollback. Use `WATCH` for optimistic locking and design idempotent operations.”**

---

## Practical Production Checklist

- **Persistence**
  - Decide RDB, AOF, or both based on durability needs.
  - Configure AOF rewrite and fsync policy.
- **Memory**
  - Set `maxmemory` and `maxmemory-policy`.
  - Monitor memory usage and eviction events.
- **Availability**
  - Choose Sentinel for simple HA or Cluster for sharding and HA.
  - Test failover scenarios and replica promotion.
- **Performance**
  - Keep values small; avoid large blobs.
  - Use pipelining and batching for high-throughput writes.
  - Monitor latency and blocked clients.
- **Data Modeling**
  - Map use cases to data types: Sorted Sets for leaderboards, Lists for queues, Streams for durable messaging, Hashes for small objects.
- **Security**
  - Enable authentication, bind to private networks, and use TLS in production.
- **Observability**
  - Collect metrics: memory, ops/sec, latency, evictions, keyspace hits/misses, replication lag.
  - Log and alert on critical events like failovers and OOM errors.

---

## Useful Commands and Config Examples

**Check memory and stats**
```bash
INFO memory
INFO stats
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lru
CONFIG SET appendonly yes
CONFIG SET appendfsync everysec
```
