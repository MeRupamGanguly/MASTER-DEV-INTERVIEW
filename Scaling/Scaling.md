# Sharding 

Sharding is horizontal partitioning: splitting a dataset across multiple machines so each node stores only a subset of the data. The goal is to increase capacity (storage and throughput) and reduce per-node load.

- Shard key — the value used to decide which shard holds a record.
- Routing — logic that sends a request to the correct shard.
- Rebalancing — moving data when nodes are added or removed.
- Hotspot — a shard receiving disproportionate traffic.
- Consistency and ordering — guarantees are usually per shard, not global.

## Sharding Strategies: 

### Hash Based Sharding Use a mathematical hash function to decide which shard stores a key. Take the key (e.g., user123).
- Compute hash(user123).
- Divide by the number of shards (N) and take the remainder: hash(user123) % N.That remainder tells you which shard to use.

Example:
- 4 shards (Shard 0–3).
- hash(user123) % 4 = 2 → goes to Shard 2.
- hash(user456) % 4 = 0 → goes to Shard 0.


### Range Based Sharding

Define ranges for shard ownership.
Example:
- Shard A: User IDs 1–1,000,000
- Shard B: User IDs 1,000,001–2,000,000

### Directory Based Sharding  

Keep a lookup table (directory) that maps each key to a shard.
Every request first checks the directory to know where to go.

A central service stores:
user123 → Shard 2
user456 → Shard 0


Example:
- Directory says: user123 is on Shard 2.
- Query goes directly to Shard 2.

### Consistent Hashing

Idea: Place shards and keys on a “ring” using a hash function.
How it works: Imagine a circle (0–360 degrees). Hash each shard and each key to a point on the circle. A key goes to the first shard clockwise from its position.

Example:
- Shard A at 90°, Shard B at 180°, Shard C at 270°.
- Key user123 hashes to 100° → goes to Shard B.
- Key user456 hashes to 260° → goes to Shard C.


# Redis Sharding
Redis sharding is built on the concept of dividing all possible keys into a fixed number of logical partitions called hash slots. There are exactly 16,384 of these slots, and this number never changes. When you store a key in Redis, the system applies a mathematical function called CRC16 to the key. The result of this calculation is a number that corresponds to one of the hash slots. This means that every key has a predetermined slot where it belongs, and the slots themselves act like containers that can be distributed across different servers.

Because the number of slots is fixed, Redis does not need to reorganize all of your data whenever you add or remove servers. Instead, it simply reassigns some of the slots to the new server or redistributes them among the remaining servers. This approach avoids what is known as a "re-hash storm," which happens in other systems when adding a new server forces every key to be recalculated and moved. In Redis, only the keys in the slots that are reassigned need to move, which makes scaling much smoother and less disruptive.

The trade-off with this design is that operations involving multiple keys can only be performed if those keys are located in the same slot. By default, keys are spread across different slots, so multi-key operations are not always possible. To solve this, Redis provides a mechanism called hash tags. By placing part of the key inside curly braces, such as {123}, you can force multiple keys to be assigned to the same slot. For example, user:{123}:profile and user:{123}:settings will both end up in the same slot, allowing you to perform operations on them together.

The choice of 16,384 slots is deliberate. It is large enough to distribute keys evenly across servers, ensuring good load balancing, but small enough to be managed efficiently. This fixed "pie" of slots makes Redis predictable and stable when scaling, since the system always knows exactly how many partitions exist and how they can be moved around. In practice, this design allows Redis clusters to grow or shrink without major disruptions, while still maintaining high performance and reliability.


# Kafka Sharding
Kafka uses sharding through a mechanism called partitioning, which is central to how it scales and distributes data. A topic in Kafka is divided into multiple partitions, and each partition is essentially a log file where messages are stored in order. When a producer sends a message, Kafka decides which partition it belongs to. If the producer provides a key, Kafka applies a hash function to that key to determine the partition. If no key is provided, Kafka can distribute messages in a round-robin fashion across partitions. This ensures that data is spread out and can be processed in parallel.

Each partition is assigned to a broker, which is a Kafka server. Brokers are responsible for storing partitions and serving them to consumers. To ensure fault tolerance, partitions are replicated across multiple brokers. One broker acts as the leader for a partition, while others act as followers. Producers and consumers interact with the leader, and followers keep copies of the data in case the leader fails. This replication strategy makes Kafka highly resilient and reliable.

Partitioning also enables Kafka to achieve parallelism. Multiple consumers can read from different partitions at the same time, which increases throughput and allows large-scale data processing. However, within a single partition, Kafka guarantees strict ordering of messages. This means that if you care about the order of events, you need to make sure all related messages go to the same partition by using a consistent key. Across partitions, ordering is not guaranteed, which is a trade-off for scalability.

Unlike Redis, where the number of hash slots is fixed at 16,384, Kafka allows you to choose the number of partitions when you create a topic. You can increase the number of partitions later, but you cannot reduce them. Increasing partitions can cause existing keys to be remapped to new partitions, which may affect ordering and consumer logic. This makes partition planning an important part of designing a Kafka system. Choosing the right number of partitions and the right keys helps balance load and ensures efficient processing.

In summary, Kafka sharding is achieved through topic partitioning. Partitions distribute data across brokers, provide fault tolerance through replication, and enable parallelism by allowing multiple consumers to process data simultaneously. The design offers scalability and resilience, but it requires careful planning of partition counts and keys to avoid uneven distribution and to preserve ordering where necessary.


# RabbitMQ Sharding

RabbitMQ approaches sharding differently than Redis or Kafka, but the idea is still about splitting data across multiple servers to scale and balance load. In RabbitMQ, sharding is usually implemented through a sharded queue. Instead of having one giant queue that could become a bottleneck, RabbitMQ allows you to create multiple smaller queues, each acting as a shard. Messages are distributed across these shards based on a hash of the routing key, so that related messages can consistently land in the same shard.

When you publish a message to a sharded queue, RabbitMQ applies a hashing function to the message’s routing key. The result of this hash determines which shard (that is, which underlying queue) the message goes into. This is similar in spirit to Redis’s CRC16 hash slots or Kafka’s partitioning, but RabbitMQ does not have a fixed number like Redis’s 16,384 slots. Instead, the number of shards is defined when you set up the sharded queue. Each shard is just a normal RabbitMQ queue, and together they form the logical sharded queue.

This design helps RabbitMQ scale horizontally. If one queue is overloaded, you can add more shards, and RabbitMQ will spread the load across them. Consumers can then connect to all shards, reading messages in parallel. This increases throughput and prevents a single queue from becoming a bottleneck. However, just like in Redis and Kafka, there are trade-offs. Ordering of messages is only guaranteed within a single shard, not across all shards. If you need strict ordering for related messages, you must ensure they share the same routing key so they hash into the same shard.

Another important point is that RabbitMQ sharding is not built into the core broker in the same way as Kafka partitions or Redis hash slots. It is implemented as a plugin called the sharding plugin. This means you need to enable and configure it explicitly. Once enabled, it provides a straightforward way to distribute messages across multiple queues without requiring the producer to know about the shards directly.

In summary, RabbitMQ sharding works by creating multiple queues under a single logical sharded queue and distributing messages among them using a hash of the routing key. This allows RabbitMQ to scale message throughput and balance load across servers. The trade-offs are similar to Redis and Kafka: ordering is only guaranteed within a shard, and careful choice of routing keys is necessary to avoid uneven distribution.

# PostgreSQL Sharding

PostgreSQL does not have sharding built into its core in the same way Redis or Kafka do, but the concept is similar: it is about splitting data across multiple servers so that no single machine becomes a bottleneck. In PostgreSQL, sharding is usually achieved through partitioning and distributed extensions such as Citus or Postgres-XL. These tools allow you to divide a large table into smaller pieces and spread those pieces across different nodes in a cluster.

The basic idea is that you choose a shard key, often a column like user_id or customer_id. PostgreSQL (with the help of extensions) applies a hashing function to that key to decide which shard the row belongs to. Each shard is stored on a different server, and together they form the logical whole of the table. This is similar to Redis’s fixed hash slots or Kafka’s partitions, except PostgreSQL does not have a fixed number like 16,384 slots. Instead, the number of shards is defined when you set up the cluster, and you can add more shards later if needed.

When you query the database, the system looks at the shard key and routes the query to the correct shard. If the query involves multiple shards, the system can run the query in parallel across them and then combine the results. This allows PostgreSQL to scale horizontally, handling much larger datasets and workloads than a single server could manage. However, just like Redis and Kafka, there are trade-offs. Operations that involve multiple shards, such as joins across different shard keys, can be slower and more complex because they require coordination across servers.

Replication and fault tolerance are also part of the design. Each shard can be replicated to other servers so that if one node fails, another can take over. This ensures reliability and high availability. The extensions that enable sharding in PostgreSQL also provide mechanisms for balancing load and redistributing shards when new servers are added to the cluster.

In summary, PostgreSQL sharding works by splitting tables into shards based on a shard key and distributing those shards across multiple servers. Queries are routed to the appropriate shard or executed in parallel across shards, and replication ensures fault tolerance. The design is conceptually similar to Redis hash slots and Kafka partitions, but PostgreSQL relies on extensions to implement it, and the number of shards is flexible rather than fixed. This makes PostgreSQL capable of scaling horizontally while still offering the rich relational features of a traditional database.


# MongoDB Sharding

MongoDB handles sharding in a way that is conceptually similar to Redis, Kafka, and PostgreSQL, but tailored to the world of document databases. Sharding in MongoDB means splitting a large collection of documents into smaller pieces, called chunks, and distributing those chunks across multiple servers, known as shards. This allows MongoDB to scale horizontally, handling very large datasets and high query loads that a single server could not manage.

When you enable sharding in MongoDB, you choose a shard key. This is a field in your documents, such as user_id or region, that MongoDB uses to decide how to distribute data. MongoDB applies a hashing or range-based strategy to the shard key to determine which chunk a document belongs to. Each chunk is then assigned to a shard. As your dataset grows, MongoDB automatically splits chunks and redistributes them across shards to keep the load balanced. This is similar to Redis’s fixed hash slots or Kafka’s partitions, but MongoDB’s chunks are dynamic—they can grow, split, and move as needed.

Queries in MongoDB are routed through a component called the mongos router. The router looks at the shard key in the query and directs the request to the correct shard. If the query involves multiple shards, the router can send the query to all relevant shards and then merge the results. This makes sharding transparent to the application: you query the database as if it were a single system, but behind the scenes, the work is distributed across many servers.

Replication also plays a key role in MongoDB sharding. Each shard is actually a replica set, meaning it has multiple copies of the data for fault tolerance. If one server fails, another replica can take over, ensuring that the system remains available. This combination of sharding for scalability and replication for reliability makes MongoDB highly resilient.

The trade-offs are similar to other systems. Choosing a good shard key is critical. A poor choice can lead to uneven distribution, where one shard gets most of the traffic (a “hot shard”), while others remain underutilized. Operations that involve multiple shards, such as joins or aggregations across different shard keys, can be slower because they require coordination across servers. Planning the shard key and understanding query patterns are essential for efficient sharding in MongoDB.

In summary, MongoDB sharding works by splitting collections into chunks based on a shard key and distributing those chunks across shards. Queries are routed through a mongos router, and replication ensures fault tolerance. This design allows MongoDB to scale horizontally while still providing the flexibility of a document database. It shares the same principles as Redis hash slots and Kafka partitions, but with dynamic chunk management that adapts as data grows.


# Application Load Balancing  
Application load balancing refers to the process of distributing incoming traffic across multiple servers at the application layer. Instead of simply balancing raw network packets, it understands the content of requests, such as HTTP headers or paths, and routes them intelligently. For example, it can send image requests to one group of servers and API requests to another. This ensures that workloads are evenly spread, improves performance, and provides high availability by preventing any single server from becoming a bottleneck.

# Elastic Load Balancing (ELB)  
Elastic Load Balancing is a service provided by cloud platforms, most notably AWS. It automatically distributes incoming application or network traffic across multiple targets, such as EC2 instances, containers, or IP addresses. The term “elastic” highlights its ability to scale dynamically with demand. As traffic increases, ELB can add more backend resources, and as traffic decreases, it can scale down. This makes it particularly useful in cloud environments where workloads fluctuate and resilience is critical.

# Nginx  
Nginx is a powerful open‑source web server that is widely used as a reverse proxy and load balancer. It can handle thousands of concurrent connections efficiently, making it ideal for high‑traffic websites. In addition to serving static content, Nginx can distribute requests across multiple backend servers, cache responses, and terminate SSL connections. Its lightweight architecture and configuration flexibility have made it one of the most popular choices for modern web infrastructure.

# Traefik  
Traefik is a modern reverse proxy and load balancer designed specifically for microservices and containerized environments. Unlike traditional tools, Traefik integrates seamlessly with orchestrators such as Kubernetes, Docker, and Consul. It automatically discovers services and configures routing without manual intervention. This dynamic behavior makes Traefik particularly well‑suited for environments where services are constantly being deployed, scaled, or removed. It also provides built‑in support for HTTPS, monitoring, and advanced routing rules.

# Reverse Proxy  
A reverse proxy is a server that sits between clients and backend servers, forwarding client requests to the appropriate backend. Unlike a forward proxy, which hides clients from servers, a reverse proxy hides servers from clients. It provides several benefits, including load balancing, caching, SSL termination, and security by shielding backend servers from direct exposure. In practice, tools like Nginx and Traefik act as reverse proxies, ensuring that requests are routed efficiently and securely to the right destination.

# API Gateway
An API Gateway is a central entry point that manages all requests coming into a system of microservices. Instead of clients calling each service directly, they send requests to the gateway, which then routes them to the correct backend service. This simplifies client interactions because the client only needs to know about the gateway, not the details of every service. It also provides a single place to enforce rules, monitor traffic, and apply security policies.

One of the most important features of an API Gateway is request routing. The gateway examines each incoming request and decides which service should handle it. This can be based on the URL path, HTTP method, or even custom headers. For example, requests to /users might go to the user service, while requests to /orders go to the order service. This routing logic makes the system flexible and allows services to evolve independently.

Another key feature is load balancing. The gateway can distribute requests across multiple instances of the same service to prevent any single server from being overloaded. This ensures better performance and reliability. Combined with health checks, the gateway can detect when a service instance is down and automatically reroute traffic to healthy ones, improving fault tolerance.

The API Gateway also provides security features. It can handle authentication and authorization, ensuring that only valid clients can access services. Common approaches include validating JSON Web Tokens (JWTs), integrating with OAuth2 providers, or enforcing API keys. By centralizing security at the gateway, individual services do not need to implement these mechanisms themselves, which reduces duplication and improves consistency.

Another powerful capability is rate limiting and throttling. The gateway can control how many requests a client can make within a certain time window. This protects backend services from being overwhelmed by sudden spikes in traffic or malicious attacks. It can also enforce quotas for different clients, ensuring fair usage of resources.

The gateway often provides caching as well. Frequently requested responses can be stored temporarily so that repeated requests are served quickly without hitting the backend service. This reduces latency and improves efficiency. For example, if many clients request the same product details, the gateway can serve the cached response instead of querying the product service every time.

Monitoring and logging are also centralized at the gateway. Because all traffic passes through it, the gateway can collect metrics such as request counts, response times, and error rates. These logs and metrics are essential for observability, helping teams detect problems, analyze performance, and plan capacity. Integrating with tools like Prometheus or ELK stacks makes this even more powerful.

Finally, an API Gateway can perform protocol translation. Clients may use HTTP, WebSockets, or gRPC, while backend services might use different protocols. The gateway can translate between them, allowing clients and services to communicate seamlessly. This makes the system more flexible and client‑friendly.

In summary, an API Gateway is much more than a router. It provides routing, load balancing, security, rate limiting, caching, monitoring, and protocol translation. By centralizing these responsibilities, it simplifies client interactions, protects backend services, and improves scalability and resilience. It is a critical component in modern microservice architectures, ensuring that systems remain efficient, secure, and easy to manage.

# Message queues
Message queues are systems that allow applications to communicate by sending messages asynchronously. Instead of one service calling another directly and waiting for a response, the sender places a message in a queue, and the receiver processes it later. This decouples services, meaning they can work independently without blocking each other.

In an event‑driven design, services react to events rather than constantly checking for changes. For example, when a user places an order, an “OrderCreated” event can be published to a queue. Other services, such as payment or shipping, subscribe to that event and act when they receive it. This makes the system more flexible and scalable because new services can be added without changing the existing ones.

Message queues also help with reliability. If a service is temporarily down, the messages remain in the queue until it comes back online. This ensures that no data is lost and that tasks are eventually completed. Systems like Kafka, RabbitMQ, and NATS are commonly used to implement these patterns.

In Go, developers often use channels for lightweight concurrency, but for distributed systems they integrate with external message queues. Worker pools can be built to consume messages in parallel, and backpressure can be managed by controlling how many messages are processed at once. This combination of Go’s concurrency model with external queues makes event‑driven systems both efficient and resilient.

In short, message queues and event‑driven design allow services to communicate asynchronously, scale independently, and remain reliable even under heavy load or partial failures. They are a cornerstone of modern distributed architectures, and Go provides the tools to implement them cleanly and effectively.
