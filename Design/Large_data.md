# Key Strategies for Handling Large Data in Golang Microservices

## 1. Data Partitioning & Sharding
Partition large datasets into smaller, manageable chunks.

Each microservice handles a shard, reducing memory load and improving scalability.

Example: A user database split by region or customer ID ranges.

## 2. Streaming & Chunked Transfers
When handling large data in Golang microservices, I rely heavily on streaming and chunked transfers rather than loading everything into memory. Go’s io.Reader and io.Writer abstractions make this natural. For example, when serving a multi‑GB file, instead of reading it all into RAM, I stream it in chunks directly to the client. This prevents memory exhaustion and keeps latency low. I’ve implemented chunked HTTP responses using Go’s standard library, which allows clients to start consuming data immediately while the rest is still being processed. This is especially useful in data pipelines where downstream services shouldn’t wait for the entire payload to be ready.

Server Side
```go
// Stream large file in chunks using io.Copy
func streamLargeFile(w http.ResponseWriter, r *http.Request) {
    file, err := os.Open("bigdata.log") //opens the file on disk. it doesn’t load the entire file into memory — it just gives you a file handle that implements io.Reader.
    if err != nil {
        http.Error(w, "File not found", http.StatusNotFound)
        return
    }
    defer file.Close()

    // Set headers for chunked transfer
    w.Header().Set("Content-Type", "application/octet-stream") // tells the client “this is raw binary data.”
    w.Header().Set("Transfer-Encoding", "chunked") // chunked tells the client “I’ll send this data in pieces (chunks), not all at once.” With chunked encoding, the server doesn’t need to know the total size beforehand. It just streams chunks until EOF.

    // Stream file directly to response writer
    if _, err := io.Copy(w, file); err != nil { // io.Copy(w, file) reads from the file (io.Reader) and writes directly to the HTTP response (io.Writer). It uses a small buffer internally (e.g., 32KB) and loops until the file ends. Each buffer read is immediately flushed to the client — so the client starts receiving data while the server is still reading the rest.
        fmt.Println("Error streaming:", err)
    }
}

```

Client Side

The client receives chunks of data progressively.
For example, if the file is 2GB, the client doesn’t wait until the server finishes reading all 2GB. It starts downloading immediately, chunk by chunk.
A client (like a browser or another microservice) makes an HTTP request to your endpoint /streamLargeFile.

```go
resp, _ := http.Get("http://localhost:8080/streamLargeFile") // This sends an HTTP GET request to your server endpoint.  The server (from your earlier code) responds by streaming the file chunk by chunk using io.Copy.

// bufio.NewScanner(resp.Body) The scanner wraps the response body and reads it line by line (by default, it splits on newline characters).
	
scanner := bufio.NewScanner(resp.Body) //  resp.Body is the response stream from the server. It’s not a complete file in memory — it’s a live stream of bytes coming in over the network.  You can read from it progressively as data arrives.
for scanner.Scan() { //  This loop runs until the stream ends (EOF).  Each iteration reads the next chunk (line) from the stream.
    fmt.Println("Chunk:", scanner.Text()) //  Prints each line (chunk) as soon as it’s read.
}

```
### Streaming Between Services

Same microservice:  
If A (producer) and B (consumer) are just different functions or modules inside one service, they can communicate directly in memory (function calls, channels, goroutines).
👉 Example: A goroutine produces data, another goroutine consumes it — no network involved.

```go
package main

import (
    "fmt"
    "time"
)

// Producer: generates data and sends it into a channel
func producer(ch chan string) {
    lines := []string{"line1", "line2", "line3"}
    for _, l := range lines {
        fmt.Println("Producing:", l)
        ch <- l // send data into channel
        time.Sleep(time.Second) // simulate delay
    }
    close(ch) // signal no more data
}

// Consumer: reads data from the channel
func consumer(ch chan string) {
    for msg := range ch { // keeps reading until channel is closed
        fmt.Println("Consumed:", msg)
    }
}

func main() {
    ch := make(chan string)

    // Run producer and consumer concurrently
    go producer(ch)
    consumer(ch) // main goroutine consumes
}

```
Different microservices:  
In a true microservices architecture, A and B are separate services (separate processes, often running on different machines/containers).
👉 Example: Service A streams logs, Service B consumes them for analytics.
They communicate over the network (HTTP, gRPC, message queues).

HTTP/REST:  
Service A exposes an endpoint (/streamData). Service B makes an HTTP request and reads the response stream.

gRPC streaming:  
Go supports gRPC with bidirectional streaming. Service A can push data continuously, and Service B consumes it.

Message queues (Kafka, NATS, RabbitMQ):  
Service A publishes messages to a topic. Service B subscribes and consumes them asynchronously.


Service Discovery:  
In microservices, services don’t hardcode IPs. They register with a service registry (like Consul, Eureka, or Kubernetes DNS).
Service B asks the registry: “Where is Service A?” and gets the address.

API Gateway:  
Both services sit behind a gateway (like Kong, Envoy, NGINX). Service B just calls gateway/api/streamData, and the gateway routes it to Service A.

Contracts (API definitions):  
They “know” each other through agreed contracts (OpenAPI specs, protobuf definitions). This ensures compatibility.


```go
// Producer: streams data line by line
func produceData(w http.ResponseWriter, r *http.Request) {
    data := strings.NewReader("line1\nline2\nline3\n") // Creates an in‑memory stream (io.Reader) containing the text:
    io.Copy(w, data) // stream to client. io.Copy reads from data (the producer’s source) and writes directly to w.
}

// Consumer: reads streamed data
func consumeData(resp *http.Response) {
    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() { // The loop runs until the stream ends (EOF).
        fmt.Println("Received:", scanner.Text())
    }
}
```



## 3. Concurrency with Goroutines
- Go’s goroutines allow parallel processing of large data sets.
- Channels coordinate communication between workers.
- Example: Processing millions of log entries concurrently without blocking.

## 4. Message Queues & Event-Driven Systems
- Use Kafka, RabbitMQ, or NATS to handle high-throughput data pipelines.
- Microservices consume data asynchronously, preventing overload.
- Ensures resilience and fault tolerance.

NATS
NATS is a very lightweight and fast messaging system. Its architecture is simple: you run a NATS server (or cluster), and clients connect to it to publish and subscribe to subjects. It is designed for low latency and millions of messages per second. By default, NATS is “fire-and-forget” (messages are not stored), but with JetStream you can add persistence and replay. The main use cases are microservices communication, IoT devices, and scenarios where speed and simplicity matter more than durability. You would use NATS when you need fast, ephemeral messaging between services, like sending notifications or coordinating tasks in a distributed system.

RabbitMQ
RabbitMQ is a traditional message broker based on the AMQP protocol. Its architecture revolves around exchanges, queues, and bindings. Producers send messages to exchanges, which route them into queues, and consumers read from those queues. RabbitMQ supports complex routing patterns, acknowledgments, retries, and message durability. It is great for transactional systems where reliability and guaranteed delivery are important. Common use cases include background job processing, order handling in e-commerce, and workflows where messages must not be lost. You would use RabbitMQ when you need flexible routing, guaranteed delivery, and strong reliability in smaller to medium-scale systems.

Kafka
Kafka is a distributed event streaming platform. Its architecture is built around topics that are partitioned and replicated across brokers. Producers write events to topics, and consumers read them with offsets, which allows replaying and processing streams of data. Kafka is designed for very high throughput and durable storage, making it suitable for big data pipelines, event sourcing, and analytics. It integrates well with tools like Spark, Flink, and Hadoop. You would use Kafka when you need to handle massive streams of data, keep them for long periods, and process them later — for example, log aggregation, clickstream analysis, or financial transaction pipelines.

Where to Use Which
- Use NATS when you want simple, fast, low-latency messaging between microservices or IoT devices, and durability is not the main concern.
- Use RabbitMQ when you need reliable message delivery with complex routing and transactional guarantees, especially in business workflows.
- Use Kafka when you need durable, scalable event streaming for analytics, big data, or event sourcing, where replay and long-term storage are critical.

## 5. Distributed Storage & Databases
- Large data is stored in NoSQL databases (MongoDB, Cassandra) or cloud-native storage (S3, GCS).
- Microservices fetch only the required subset of data.
- Avoids monolithic database bottlenecks.

## 6. Caching & Compression
- Redis or Memcached for caching frequently accessed large data.
- Compression (gzip, snappy) reduces payload size during transfers.
- Improves performance and reduces network bandwidth usage.

## 7. API Gateway & Pagination
- API Gateway enforces rate limiting and pagination for large queries.
- Clients receive data in smaller batches instead of overwhelming services.

## Websockets
- gRPC and WebSockets both enable real-time, bidirectional communication, but they serve different purposes: gRPC is optimized for service-to-service RPC with strong typing and code generation, while WebSockets are better suited for client-facing applications needing flexible, persistent connections.

# Send millions of rows 
Send millions of rows reliably by streaming them in chunks (NDJSON/CSV), using bulk/batch endpoints or async ingestion (message queue), and designing both client and server for backpressure, idempotency, and efficient DB bulk writes.

## Streaming NDJSON or CSV
What  
Send rows as a continuous stream where each line is one record (NDJSON) or one CSV row.

Why  
Streaming keeps memory usage low on both client and server and lets the server begin processing immediately instead of waiting for the entire payload.

How  
Client opens an HTTP POST and writes lines to the request body; server reads the request body line by line and processes each record as it arrives. Use chunked transfer encoding or keep the connection open for long uploads.

```go
// client: stream NDJSON using io.Pipe
pr, pw := io.Pipe()
req, _ := http.NewRequest("POST", url, pr)
go func() {
  enc := json.NewEncoder(pw)
  for _, r := range rows {
    enc.Encode(r)
  }
  pw.Close()
}()
http.DefaultClient.Do(req)

// server: read stream line by line
scanner := bufio.NewScanner(r.Body)
for scanner.Scan() {
  var row Row
  json.Unmarshal(scanner.Bytes(), &row)
  process(row)
}

````

Operational notes

Use small, bounded buffers and sync.Pool for temporary buffers.

Support partial acknowledgements or checkpoints for very long streams.

Protect against slow clients by setting read timeouts and limits.

Monitoring  
Track rows/sec, stream duration, memory usage, and error rate for streamed requests.

Full answer to say  
“I use NDJSON streaming for large imports: the client writes one JSON line per record to an HTTP POST and the server reads line by line with a scanner. This keeps memory low, lets processing start immediately, and supports resumable checkpoints. I add timeouts and limits to protect the service from slow or malicious clients.”


## Bulk Batch Endpoints
What  
Group rows into fixed-size batches (typical sizes 1k–100k rows) and send each batch in a single request.

Why  
Databases and storage engines are far more efficient with bulk operations; batching reduces per-row overhead, network round trips, and transaction cost.

How  
Client accumulates rows into a batch, optionally compresses the payload (gzip), and POSTs it. Server receives the batch and uses the database bulk API (COPY, bulk loader, or batched prepared statements) to write efficiently.

```go
// client: send gzip-compressed batch
var buf bytes.Buffer
gw := gzip.NewWriter(&buf)
json.NewEncoder(gw).Encode(batch)
gw.Close()
http.Post(url+"/ingest/batch", "application/json", &buf)

// server: receive and write batch
var batch []Row
json.NewDecoder(r.Body).Decode(&batch)
writeBulkToDB(batch) // use COPY or bulk insert

````
Operational notes

Tune batch size by measuring DB write latency, memory, and GC.

Compress batches to reduce network I/O.

Keep transactions bounded to avoid long locks; consider staging tables.

Monitoring  
Measure batch latency, DB write time, batch success rate, and memory per worker.

Full answer to say  
“I implement fixed-size batch endpoints where the client sends compressed batches of rows and the server uses the DB bulk API like Postgres COPY. I tune batch size based on DB write latency and memory, keep transactions bounded, and use staging tables when necessary to avoid impacting OLTP traffic.”

## Asynchronous Ingestion with Message Queue
What  
Publish rows or events to a durable message system (Kafka, NATS JetStream, RabbitMQ); consumers read from the queue and perform bulk writes.

Why  
Queues decouple producers from consumers, absorb traffic spikes, provide durability and replay, and let you scale consumers independently of producers.

How  
Producer publishes messages to a topic. Consumer groups read partitions in parallel, aggregate messages into batches, and write those batches to the database. Use partitioning keys to preserve ordering where needed.

```go
// producer: publish events
for _, row := range rows {
  producer.Publish(topic, encode(row))
}

// consumer: read, batch, write
for msg := range consumer.Messages() {
  batch = append(batch, decode(msg))
  if len(batch) >= 1000 {
    writeBulkToDB(batch)
    batch = batch[:0]
  }
}

````
Operational notes

Monitor consumer lag and autoscale consumers when lag grows.

Choose retention and partitioning strategy to balance throughput and replay needs.

Use exactly-once or at-least-once semantics depending on requirements and implement deduplication if needed.

Monitoring  
Track producer throughput, consumer lag, partition skew, and end-to-end latency from publish to DB commit.

Full answer to say  
“I use durable queues like Kafka to decouple ingestion from storage: producers publish events and consumer groups scale to drain partitions and perform bulk DB writes. This absorbs spikes, enables replay, and lets us scale consumers independently to match DB throughput.”



## Backpressure Mechanisms
What  
Mechanisms that slow or reject producers when downstream systems are overloaded.

Why  
Backpressure prevents cascading failures and keeps the system stable under spikes or slowdowns.

How  
Return HTTP 429 with Retry-After for overloaded endpoints, implement streaming ACKs so the server signals progress, throttle producers at the client or queue client, and use exponential backoff with jitter for retries. For queues, apply producer-side rate limits or broker quotas.

```bash
// HTTP: respond with 429 when queue lag or DB latency exceeds threshold
HTTP/1.1 429 Too Many Requests
Retry-After: 30

```
Operational notes

Define clear thresholds (consumer lag, DB write latency) that trigger backpressure.

Combine short-term buffering with rejection to avoid unbounded memory growth.

Provide observability so clients can adapt their send rate.

Monitoring  
Alert on 429 rate, queue lag, DB write latency, and number of throttled clients.

Full answer to say  
“I implement backpressure by monitoring consumer lag and DB latency and returning 429s with Retry-After when thresholds are exceeded. For streaming, I use ACKs or checkpoints so clients can slow down or resume. This prevents cascading failures and keeps the ingestion pipeline stable.”


## Idempotency and Ordering
What  
Guarantee that retries do not create duplicate rows and decide how strict ordering must be for business correctness.

Why  
Network failures and retries are normal; idempotency prevents duplicate data and inconsistent state. Ordering is required only when business logic depends on sequence.

How  
Attach batch IDs or sequence numbers to each batch or row. Deduplicate on the server using those IDs or use idempotent upserts. For ordering, partition by key (user ID, tenant) and ensure a single consumer processes each partition in order. If global ordering is required, accept lower throughput or use a single ordered partition.

```sql
-- Postgres upsert example
INSERT INTO events (id, payload) VALUES (...)
ON CONFLICT (id) DO NOTHING;

```

Operational notes

Use idempotency keys for client retries and store recent keys with TTL to deduplicate.

For per-key ordering, map keys to partitions and process partitions sequentially.

Document ordering guarantees for API consumers.

Monitoring  
Track duplicate detection rate, idempotency key store size, and ordering violations.

Full answer to say  
“I require each batch to carry an idempotency key and sequence numbers when needed. The ingestion service deduplicates by key before applying bulk writes. For ordering-sensitive data I partition by key and process each partition in order; for everything else I favor higher throughput with per-key ordering.”

## Efficient Database Bulk Writes
What  
Use database bulk APIs and load strategies to minimize transaction overhead and locking during large imports.

Why  
Bulk APIs like Postgres COPY or MySQL bulk load are orders of magnitude faster than per-row inserts and reduce CPU and lock contention.

How  
Write to staging tables with minimal indexes, use COPY or bulk loaders, then run controlled reindexing or merge into production tables. Use partitioning or sharding to parallelize writes. Keep transactions bounded and monitor replication lag.

```sql

-- Postgres COPY from STDIN for high throughput
COPY staging_table (col1, col2) FROM STDIN WITH (FORMAT csv);

```
Operational notes

Disable or defer nonessential indexes during large loads and rebuild afterward.

Use partitioned tables to parallelize ingestion and reduce contention.

Monitor DB CPU, I/O, lock waits, and replication lag; tune batch size accordingly.

Monitoring  
Measure DB write throughput, transaction duration, lock wait time, and replication lag.

Full answer to say  
“For large imports I write to a staging table using COPY or bulk loaders with minimal indexes, then run controlled reindexing or merge steps. I partition or shard writes to parallelize load and keep transactions bounded to avoid long locks. I monitor DB metrics closely and tune batch size and concurrency to maximize throughput without impacting OLTP traffic.”



# Handling 2 million requests per second 

Handling 2 million requests per second is an extreme scale — think Google, Facebook, or Netflix levels. To get anywhere near that in a Go microservice, you need to combine application-level optimizations with infrastructure-level scaling. 

A single optimized Go service might handle 50k–100k RPS on powerful hardware.

To reach 2M RPS, you’d need 20–40 replicas (depending on workload complexity).

With global distribution (multiple regions), you scale horizontally across clusters.

## Application-Level Optimizations (Inside Go)

### Efficient Concurrency
What it means: Use goroutines for parallelism but control concurrency so the runtime, CPU, and external resources (DB, network) aren’t overwhelmed.
Why it matters: Unbounded goroutine creation can spike memory, increase scheduler overhead, and exhaust external resources.
Pattern: Worker pool — a fixed number of workers consume tasks from a queue, providing backpressure and predictable resource use.

```go
type Job func()

func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
  defer wg.Done()
  for job := range jobs {
    job()
  }
}

func startPool(n int, jobs chan Job) {
  var wg sync.WaitGroup
  for i := 0; i < n; i++ {
    wg.Add(1)
    go worker(i, jobs, &wg)
  }
  wg.Wait()
}

```

### Avoid Heavy Allocations
What it means: Reduce garbage collection (GC) pressure by reusing memory and avoiding temporary allocations.
Why it matters: GC pauses and high allocation rates increase tail latency at scale.
Tools & patterns: sync.Pool for reusable buffers; preallocate slices; prefer streaming to avoid building large in-memory objects.

```go
var bufPool = sync.Pool{New: func() any { return make([]byte, 32*1024) }}

func handle() {
  b := bufPool.Get().([]byte)
  defer bufPool.Put(b)
  // use b as temporary buffer
}

```

### Fast Networking
What it means: Use protocols and libraries that reduce connection overhead and serialization cost.
Why it matters: Network and serialization dominate latency at high RPS.
Options:

HTTP/2 or gRPC (multiplexed streams, binary protobufs) for service-to-service calls.

fasthttp for ultra-low-latency HTTP servers when you control the stack.

Use keep-alive, connection pooling, and TLS offload at the edge.

Small note: gRPC gives strong typing and code generation; HTTP/2 reduces TCP churn.

### Stateless Services
What it means: Keep request handling free of local, sticky state so any replica can serve any request.
Why it matters: Statelessness enables simple horizontal scaling, rolling updates, and fault tolerance.
Where state goes: Redis or distributed caches for sessions; durable stores (Cassandra, CockroachDB, S3) for persistent data.
```go
// store session id in Redis, not in-memory
rdb.Set(ctx, sessionID, userData, 30*time.Minute)

```

## Infrastructure-Level Scaling

### Horizontal Scaling
What — Run many identical stateless replicas of your Go service and scale them out across nodes and regions.
Why — A single machine cannot handle millions of RPS; capacity comes from many machines working in parallel. Statelessness makes scaling simple and safe.
How — Use an orchestrator (Kubernetes, Nomad) to manage replicas, health checks, and rolling updates. Autoscale based on real metrics (RPS per pod, CPU, latency).
Small example — Kubernetes HPA concept (pseudo YAML snippet):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-service-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: my-service
  minReplicas: 10
  maxReplicas: 1000
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

```
### Load Balancing and Geo Distribution
What — Distribute incoming traffic evenly and route users to the nearest healthy region.
Why — Even distribution prevents hotspots and reduces latency by serving users from nearby clusters.
How — Use Envoy, NGINX, or HAProxy at the cluster edge; use global load balancers or anycast DNS for multi‑region routing. Implement health checks and weighted routing for gradual rollouts.
Small example — Envoy sits in front of pods and performs circuit breaking, retries, and local load balancing; global LB routes traffic to the closest region and fails over if a region is unhealthy.
Interview tip — Mention connection reuse (keep‑alive), TLS termination at the edge, and how you avoid head‑of‑line blocking with HTTP/2 or gRPC.

### API Gateway
What — A centralized entry point that enforces cross‑cutting policies: auth, rate limiting, caching, and request shaping.
Why — Gateways reduce load on services by rejecting or caching requests early and by collapsing duplicate requests. They also provide a single place for observability and security.
How — Implement rate limits per API key or IP, response caching for idempotent endpoints, and request collapsing (coalescing) so concurrent identical requests are served by one backend call. Use token buckets or leaky buckets for throttling.
Small example — If 100 clients request the same heavy report, the gateway returns a cached response or collapses them into one backend call and fans out the result.
Interview tip — Explain trade‑offs: aggressive caching reduces load but increases staleness; collapsing adds complexity but saves backend cycles.

### Caching and CDNs
What — Push static and hot dynamic content to caches close to users. Use CDNs for static assets and Redis/Memcached for hot keys.
Why — Caches reduce origin load, lower latency, and absorb traffic spikes. A well‑tuned cache can cut backend RPS by orders of magnitude.
How — Cache at multiple layers: CDN at the edge, API gateway cache for HTTP responses, and in‑region Redis for session or hot object caching. Use TTLs and cache invalidation strategies (write‑through, write‑behind, or explicit invalidation).
Small example — Serve images and JS from CDN; store user profile lookups in Redis with a 60s TTL to avoid repeated DB hits.
Interview tip — Discuss cache invalidation patterns and how you measure cache hit ratio and its impact on backend load.

### Data Tier and Asynchronous Backbone
What — Scale storage and offload work using sharded databases and message brokers.
Why — Databases are common bottlenecks; sharding and distributed stores spread load. Message queues decouple producers from consumers and smooth traffic spikes.
How —

Database scaling: shard by key (user ID, tenant), use read replicas for reads, and choose distributed databases (CockroachDB, Cassandra) when you need horizontal consistency and availability.

Message queues: use Kafka for durable, replayable streams and analytics; use NATS or RabbitMQ for low‑latency pub/sub and task queues. Offload non‑critical work (emails, analytics, thumbnails) to queues so the request path stays fast.
Small example — Write path: service writes minimal transaction to DB and publishes an event to Kafka; background consumers enrich data and update secondary stores.
Interview tip — Explain how you choose retention and partitioning strategy for Kafka, and how you handle consumer lag and backpressure.

## Observability & Reliability
Prometheus (metrics): Prometheus pulls (scrapes) /metrics endpoints on services at intervals, stores time‑series data with labels, and supports alerting and queries (PromQL). Use it to monitor latency, throughput, error rates. 

Loki (logs): Loki stores compressed log chunks and indexes only labels (not full log text), making it cost‑efficient for large volumes; you query logs by labels and time range in Grafana. 

Grafana (visualization): Grafana connects to Prometheus and Loki to build dashboards and correlate metrics ↔ logs; it’s the UI and alerting front end.

Promtail is the lightweight agent that collects logs from your hosts or containers and pushes them to Loki. It tails files, reads systemd journals, or receives logs over syslog, enriches them with labels, optionally processes them with pipeline stages, and forwards them to Loki for storage and later querying in Grafana.

expose Prometheus metrics Prometheus scrapes /metrics periodically; tune scrape interval and labels for cardinality. 
```go
import (
  "net/http"
  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

var reqs = prometheus.NewCounterVec(
  prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests"},
  []string{"path","status"},
)

func init() { prometheus.MustRegister(reqs) }

func handler(w http.ResponseWriter, r *http.Request) {
  // business logic...
  reqs.WithLabelValues(r.URL.Path, "200").Inc()
  w.Write([]byte("ok"))
}

func main() {
  http.Handle("/metrics", promhttp.Handler()) // Prometheus scrapes this
  http.HandleFunc("/", handler)
  http.ListenAndServe(":8080", nil)
}

```
OpenTelemetry tracing
```go
// simplified: set up OTLP exporter and instrument HTTP handlers
import (
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
  "go.opentelemetry.io/otel/sdk/trace"
)

exp, _ := otlptracehttp.New(context.Background())
tp := trace.NewTracerProvider(trace.WithBatcher(exp))
otel.SetTracerProvider(tp)
tr := otel.Tracer("orders-service")
ctx, span := tr.Start(context.Background(), "handleOrder")
defer span.End()

```

push logs to Loki (HTTP)
```go
// send a simple log line to Loki push endpoint
import (
  "bytes"
  "net/http"
)
payload := `{"streams":[{"labels":"{job=\"svc\"}","entries":[{"ts":"2026-02-05T14:00:00Z","line":"order processed"}]}]}`
http.Post("http://loki:3100/loki/api/v1/push","application/json", bytes.NewBufferString(payload))

```
