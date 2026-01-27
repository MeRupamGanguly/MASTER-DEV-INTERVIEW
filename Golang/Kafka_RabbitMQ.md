
RabbitMQ is a traditional message broker designed for discrete task distribution and complex routing.

Kafka, conversely, is a distributed event streaming platform built on an append-only commit log.

RabbitMQ follows a "smart broker, dumb consumer" model. The broker manages the state of every message, tracking which consumer has acknowledged which task and then deleting the message once it is processed.

Kafka uses a "dumb broker, smart consumer" model. It treats data as a continuous stream of events that stay on disk for a set retention period. The consumer is responsible for keeping track of its own position (the offset) in the log.

Kafka is the clear winner for raw throughput. Because it uses sequential disk I/O and zero-copy data transfer, a single Kafka cluster can handle millions of messages per second, making it the industry standard.

RabbitMQ is no slouch, comfortably handling tens of thousands of messages per second, but it can struggle at hyper-scale because managing individual message acknowledgments at the broker level becomes a bottleneck. However, 

RabbitMQ often offers lower latency for individual, low-volume messages because it pushes them directly to consumers as soon as they arrive, whereas Kafka consumers typically pull data in batches.

RabbitMQ shines with complex routing (using exchanges like headers, fan-out, or topics) and built-in features like Priority Queues and Dead Letter Exchanges for handling failed tasks. Kafka offers superior ordering guarantees (within a partition) and fault tolerance through native replication. Its most powerful feature is Message Replay; if a bug is found in your processing logic, We can simply "rewind" your Kafka consumer to re-process the last week of data, a task that is nearly impossible in RabbitMQ once messages have been acknowledged and deleted.

Kafka uses topics to send and receive messages, which can be thought of as analogous to queues in traditional messaging systems. However, in Kafka, we call them topics. Each topic can have multiple partitions. Each partition contains a subset of the messages for a specific topic. Kafka ensures that messages are ordered within a partition, but doesn't guarantee ordering across different partitions in the same topic.

Partitions allow Kafka to scale horizontally. This means that each partition can be processed in parallel by multiple machines, making Kafka capable of handling large-scale data streams.

Kafka retains messages even after they are consumed, and multiple consumers can read the same message at different times.

Multiple consumers can form a consumer group. Kafka ensures that only one consumer from the group consumes each message within a topic partition. If there are N partitions for a topic and N consumers in the group, each consumer will be assigned to one partition, and each partition will be read by only one consumer.

Kafka follows a Leader-Follower model for partition replication. The leader broker is responsible for handling both read and write requests for a partition. Follower brokers replicate data from the leader. If the leader broker crashes, one of the follower brokers is elected to become the new leader, ensuring high availability and fault tolerance. Followers do not handle client requests directly, they simply replicate data from the leader.

```go
p, err := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers":  "localhost:9092",
    "acks":               "all",              // durability
    "enable.idempotence": true,               // EOS safety
    "compression.type":   "lz4",              // better throughput
    "linger.ms":          10,                 // batch messages for efficiency
    "batch.num.messages": 1000,               // tune batching
    "message.timeout.ms": 60000,              // fail fast if broker unreachable
})

c, err := kafka.NewConsumer(&kafka.ConfigMap{
    "bootstrap.servers":   "localhost:9092",
    "group.id":            "payments-service",
    "auto.offset.reset":   "earliest",
    "enable.auto.commit":  false,             // manual offset control
    "max.poll.interval.ms": 300000,           // allow long processing
    "fetch.min.bytes":     1024,              // batch fetch tuning
    "fetch.max.bytes":     52428800,          // cap memory usage
})
```
bootstrap.servers → The Kafka broker(s) your producer connects to. "localhost:9092" means it’s running locally. In production, you’d list multiple brokers for resilience.

acks=all → The producer waits until all in-sync replicas acknowledge the message. This guarantees durability (no data loss if the leader crashes).

enable.idempotence=true → Ensures exactly-once semantics for the producer. Prevents duplicates if retries happen. Critical for financial or transactional systems.

compression.type=lz4 → Compresses messages before sending. LZ4 is fast and efficient, reducing network + disk usage, improving throughput.

linger.ms=10 → Waits up to 10ms before sending a batch. This allows multiple messages to be grouped together, improving efficiency.

batch.num.messages=1000 → Maximum number of messages per batch. Larger batches = better throughput, but more latency.

message.timeout.ms=60000 → If a message cannot be delivered within 60 seconds, fail it. Prevents infinite retries and lets We handle errors quickly.

bootstrap.servers → Same as producer: list of brokers to connect to.

group.id=payments-service → Identifies the consumer group. All consumers with the same group ID share partitions. Here, it’s named after the service.

auto.offset.reset=earliest → If no committed offset exists, start reading from the earliest available message. Useful for new consumers that want the full history.

enable.auto.commit=false → Disables automatic offset commits. We commit offsets manually after successful processing. This prevents “false progress” if a message fails.

max.poll.interval.ms=300000 (5 minutes) → Maximum time between Poll() calls before Kafka considers the consumer dead. Setting it high allows long processing tasks without triggering a rebalance.

fetch.min.bytes=1024 (1 KB) → Broker waits until at least 1 KB of data is available before responding. Helps batch small messages together for efficiency.

fetch.max.bytes=52428800 (50 MB) → Maximum data per fetch request. Prevents a single consumer from being overwhelmed by huge batches.


## Security

SSL (TLS) → encrypts traffic between client and broker.

SASL (Simple Authentication and Security Layer) → provides authentication (username/password, Kerberos, OAuth, etc.).

Together, SASL/SSL ensures both secure communication and identity verification.

security.protocol=SASL_SSL  
This tells the Kafka client to use SASL authentication over an SSL/TLS encrypted connection.

SSL ensures the traffic between client and broker is encrypted.

SASL provides authentication (so the broker knows who We are).
Together, this prevents eavesdropping and unauthorized access.

sasl.mechanisms=PLAIN  
Defines the SASL mechanism used for authentication.

PLAIN → Username/password in plain text (but sent securely inside SSL).

Alternatives: SCRAM-SHA-256, SCRAM-SHA-512, GSSAPI (Kerberos), OAUTHBEARER.
In production, SCRAM or Kerberos is often preferred for stronger security.

sasl.username / sasl.password  
Credentials provided by the Kafka cluster administrator.

These are used to authenticate the client.

In Confluent Cloud, for example, We get an API key and secret here.

ssl.ca.location=/etc/ssl/certs/ca.pem  
Path to the Certificate Authority (CA) file that verifies the broker’s SSL certificate.

Prevents man‑in‑the‑middle attacks.

Ensures the broker We connect to is trusted.

In production, this file is usually provided by your Kafka cluster admin or cloud provider.

- Create Privete key by OPENSSL command choose algoriyh as RSA.
- Generate Certificates. Need to Sign those certificates by CA.
- Kafka Broker need to onfigured to support SSL/TLS. For this we can modify the server.properties file.
- When we call kafka.NewProducer(&kafka.ConfigMap{...} and kafka.NewConsumer(&kafka.ConfigMap{..} we need to pass security protocol ssl/tls, CA certificate location, Client certificate location, Client private key location, client pasword etc.
- both the Kafka server and Kafka clients (e.g., producers, consumers) need their own certificates and private keys if we are enabling mutual TLS (mTLS), which is highly recommended for secure production environments. 
- On the Kafka Client and server (Producer/Consumer) Private Key, Signed Certificate, Keystore (with private key + certificate), Truststore (with server CA certificate) needed.


## Producer Example with SASL/SSL
```go
package main

import (
    "fmt"
    "log"

    "github.com/confluentinc/confluent-kafka-go/kafka"
)

func main() {
    p, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers": "my-secure-broker:9093",
        "security.protocol": "SASL_SSL",          // Use SASL over SSL
        "sasl.mechanisms":   "PLAIN",             // Mechanism (PLAIN, SCRAM, etc.)
        "sasl.username":     "my-username",       // Provided by broker admin
        "sasl.password":     "my-password",       // Provided by broker admin
        "ssl.ca.location":   "/etc/ssl/certs/ca.pem", // CA cert to verify broker
        "acks":              "all",
        "enable.idempotence": true,
    })
    if err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }

    // Delivery reports
    go func() {
        for e := range p.Events() {
            switch ev := e.(type) {
            case *kafka.Message:
                if ev.TopicPartition.Error != nil {
                    fmt.Printf("Delivery failed: %v\n", ev.TopicPartition.Error)
                } else {
                    fmt.Printf("Delivered to %v\n", ev.TopicPartition)
                }
            }
        }
    }()

    topic := "secure-topic"
    msg := &kafka.Message{
        TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
        Key:   []byte("user123"),
        Value: []byte("Secure message with SASL/SSL"),
    }

    p.Produce(msg, nil)
    p.Flush(5000)
}

```

## Consumer Example with SASL/SSL
```go
package main

import (
    "fmt"
    "log"

    "github.com/confluentinc/confluent-kafka-go/kafka"
)

func main() {
    c, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": "my-secure-broker:9093",
        "security.protocol": "SASL_SSL",
        "sasl.mechanisms":   "PLAIN",
        "sasl.username":     "my-username",
        "sasl.password":     "my-password",
        "ssl.ca.location":   "/etc/ssl/certs/ca.pem",
        "group.id":          "secure-group",
        "auto.offset.reset": "earliest",
    })
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }

    c.SubscribeTopics([]string{"secure-topic"}, nil)

    for {
        ev := c.Poll(100)
        if ev == nil {
            continue
        }
        switch e := ev.(type) {
        case *kafka.Message:
            go func(msg *kafka.Message) {
                err := processPayment(msg.Value)
                if err != nil {
                    // Send to DLQ or retry queue
                    log.Printf("Failed processing offset %d: %v", msg.TopicPartition.Offset, err)
                } else {
                    // Commit offset only after success
                    _, err := c.CommitMessage(msg)
                    if err != nil {
                        log.Printf("Commit failed: %v", err)
                    }
                }
            }(e)
        case kafka.Error:
            fmt.Printf("Error: %v\n", e)
        }
    }
}
```

Poll() is the method We call on a Kafka consumer (or producer) to retrieve events from Kafka.

Poll() is the Kafka client’s way of fetching events and keeping the consumer alive. It’s single-threaded, must be called regularly, and is the entry point for messages, errors, and rebalance notifications. In Go, We wrap it in a loop and handle each event type appropriately.

Events: These events can be:
    A new message (*kafka.Message) from a topic partition.
    A rebalance event (partitions assigned or revoked).
    An error (kafka.Error).
    Delivery reports (for producers).

## Reliability & Data Integrity (Producer):
Reliability in Kafka means making sure that once We send a message, it doesn’t get lost or duplicated. The producer achieves this by waiting for acknowledgements (acks=all) from all replicas in the cluster. In Go, We configure this in the producer settings. We also enable idempotence, which ensures retries don’t create duplicates.

When We set acks=all, Kafka waits until the leader and all in-sync replicas (ISR) have written the message before confirming. This guarantees that even if the leader dies immediately after acknowledging, another replica already has the data and can take over.

```go
config := &kafka.ConfigMap{
    "acks": "all",              // Wait for all replicas
    "enable.idempotence": true, // Prevent duplicates
    "retries": 5,               // Retry transient errors
}

go func() {
    for e := range p.Events() {
        if ev, ok := e.(*kafka.Message); ok && ev.TopicPartition.Error != nil {
            log.Printf("Persistent Failure: %v", ev.TopicPartition.Error)
        }
    }
}()

```
This goroutine continuously listens for delivery reports. If a message fails permanently, We log it or trigger alerts. That’s how We guarantee reliability.

## Exactly-Once Semantics (Transactions)

Normally, when We send data to Kafka, there are two risks:

Duplicates – If the producer retries after a network hiccup, the same message might be written twice.

Loss – If the producer crashes after consuming but before committing, Kafka might think the message was never processed.

EOS solves this by combining idempotent producers with transactions. Think of it like a bank transfer: either the money moves once or not at all. Kafka uses a Transaction Coordinator broker to manage this two-phase commit process.

So even if We retry, crash, or restart, Kafka ensures the final state looks as if each message was processed exactly once.

When We use transactions in Kafka, the messages We produce are not immediately visible to consumers. Instead, they are written into the transactional log on the broker, but marked as pending until We call CommitTransaction().

```go
// Initialize transactions for this producer // This tells Kafka We want to use transactions. It sets up a unique transactional ID for the producer.
err := p.InitTransactions(nil)
if err != nil {
    log.Fatalf("Failed to initialize transactions: %v", err)
}

// Begin a new transaction
err = p.BeginTransaction()
if err != nil {
    log.Fatalf("Failed to begin transaction: %v", err)
}

// Produce a message inside the transaction
msg := &kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
    Key:   []byte("order123"),
    Value: []byte("payment processed"),
}
p.Produce(msg, nil) // We send your message as usual, but it’s not visible to consumers until the transaction is committed.

// Commit consumer offsets atomically with the message
err = p.SendOffsetsToTransaction(nil, offsets, consumerMetadata, nil) //  This is the magic part. We commit the consumer’s offsets inside the same transaction. That way, Kafka knows: “This consumer processed up to offset X, and also produced message Y.”
if err != nil {
    // Abort transaction: discard pending messages 
    p.AbortTransaction(nil) // Those records are flagged as aborted and consumers won’t see them.
    log.Fatalf("Failed to send offsets: %v", err)
}

// Commit the transaction (message + offsets together)
err = p.CommitTransaction(nil) // Finalizes the transaction. Both the message and the offsets are atomically committed. If We crash before this, Kafka discards the transaction, so no duplicates or partial commits happen.
if err != nil {
    log.Fatalf("Failed to commit transaction: %v", err)
}

```

## High-Performance Consumer (Parallelism)

We finish page 10, so We put a bookmark on page 11. Next time, We start from page 11. That’s exactly what Kafka offsets are: bookmarks in the log.

If We mark page 11 before actually finishing page 10, We’ll skip content. If We forget to mark and restart, you’ll reread content.

So We must commit offsets only after all earlier messages are truly processed.

Kafka stores messages in partitions (like ordered logs). Each consumer in a group is assigned one or more partitions.

Within a partition, messages are strictly ordered by offsets (like line numbers in a file).

A consumer must keep track of the last offset it has processed so that if it restarts, it knows where to continue.

This tracking is called committing offsets.

Single Partition Case → A commit is basically an integer offset (e.g., 42).
This means: “I’ve processed everything up to offset 41, so start me at 42 next time.”

Multiple Partitions Case → A commit is an array (list) of offsets, one per partition.
Because a consumer group member may own multiple partitions at once.
Kafka needs to know the committed offset for each partition separately.

```go
// Commit a single message offset
consumer.CommitMessage(msg)

// Commit multiple offsets at once
offsets := []kafka.TopicPartition{
    {Topic: &topic, Partition: 0, Offset: kafka.Offset(42)},
    {Topic: &topic, Partition: 1, Offset: kafka.Offset(105)},
}
_, err := consumer.CommitOffsets(offsets)
if err != nil {
    log.Printf("Commit failed: %v", err)
}

```
Now imagine We process messages in parallel (using multiple workers)

    Worker A gets offset 10.
    Worker B gets offset 11.
    Worker B finishes first.

If We commit offset 11 immediately, Kafka thinks offset 10 is also done—but it’s not! That’s unsafe.

We need a coordinator that tracks which offsets are finished.
It waits until all earlier offsets are done. Then it commits the lowest continuous offset (the “low watermark”).

    Imagine :
    Worker A is processing offset 10. 
    Worker B is processing offset 11. 
    Worker C is processing offset 12.

The coordinator can only safely commit up to the lowest continuous offset that’s finished. If 10 is not done, even if 11 and 12 are finished, We cannot commit 12.

We must wait until 10 is finished, then commit 13 (meaning “I’ve processed up to 12”). This way, no gaps are left unprocessed.


## Log Compaction
Kafka will keep at least the most recent message for each key.
Older messages with the same key are eventually deleted during compaction.
This makes the topic behave more like a key-value store than a pure event log.

Key=user123, Value=email=old@example.com
Key=user123, Value=email=new@example.com

After compaction, only the second record remains. Kafka normally keeps all messages for a certain retention period (say 7 days). But sometimes We don’t care about the full history—We only want the latest value per key.

In terminal This tells Kafka: “Compact this topic, keep only the latest value per key.”
```bash
bin/kafka-topics.sh --create \
  --topic user-profiles \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 2 \
  --config cleanup.policy=compact

```

## Consumer Lag
Consumer lag is simply how far behind your consumer is compared to the latest data in Kafka. If lag grows, it means your consumer can’t keep up.
In Go, We can check lag like this:

```go
low, high, _ := consumer.QueryWatermarkOffsets("my-topic", partition, 5000)
fmt.Printf("Latest offset: %d, Current offset: %d\n", high, currentOffset)
```
Here, high is the broker’s latest offset, and currentOffset is where your consumer is. The difference is the lag. If lag is large, We either need more partitions, faster processing, or better parallelism.

Kafka consumer groups divide partitions among consumers.

When a consumer joins, leaves, or becomes unresponsive, Kafka rebalances the group.

During rebalance, partitions may be reassigned → consumption pauses briefly.

The cooperative-sticky strategy (newer default in many setups) minimizes disruption by only moving partitions that must be reassigned.

```go
package main

import (
    "fmt"
    "log"

    "github.com/confluentinc/confluent-kafka-go/kafka"
)

func main() {
    c, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "group.id":          "rebalance-demo",
        "auto.offset.reset": "earliest",
        // Cooperative-sticky strategy
        "partition.assignment.strategy": "cooperative-sticky",
    })
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }

    c.SubscribeTopics([]string{"my-topic"}, nil)

    for {
        ev := c.Poll(100) // Poll() loop → Continuously fetches messages and rebalance events.
        if ev == nil {
            continue
        }

        switch e := ev.(type) {
        case *kafka.Message:
            fmt.Printf("Consumed message: %s from %s [%d] at offset %d\n",
                string(e.Value), *e.TopicPartition.Topic, e.TopicPartition.Partition, e.TopicPartition.Offset)
            c.CommitMessage(e)

        case kafka.AssignedPartitions:
            // Triggered when partitions are assigned during rebalance
            fmt.Printf("Partitions assigned: %v\n", e.Partitions)
            c.Assign(e.Partitions)

        case kafka.RevokedPartitions:
            // Triggered when partitions are revoked during rebalance
            fmt.Printf("Partitions revoked: %v\n", e.Partitions)
            c.Unassign()

        case kafka.Error:
            fmt.Printf("Error: %v\n", e)
        }
    }
}

```

Backpressure happens when consumers can’t keep up with producers. Kafka is pull-based, so if We stop polling, data just waits in the broker. But if We stop too long, Kafka thinks your consumer is dead.
This tells Kafka to stop sending data for that partition, while your consumer still sends heartbeats. Once you’re ready, We call Resume().
```go
consumer.Pause([]kafka.TopicPartition{{Topic: &topic, Partition: 0}})
```

