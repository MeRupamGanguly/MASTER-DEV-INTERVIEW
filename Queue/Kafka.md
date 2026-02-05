
# Kafka
- Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant, real-time data pipelines and applications.
- Unlike RabbitMQ (which is a message broker), Kafka is fundamentally a commit log where events are stored durably and can be replayed.
- We can Use Kafka when: Message Replay is needed: If a service crashes or you deploy a bug, you can "rewind" the offset and re-process the last 24 hours of data.
- Kafka can Handles millions of events per second across distributed clusters.
- Kafka Stores events on disk with replication, ensuring no data loss.
- Kafka treats data as a continuous stream of events that stay on disk for a set retention period. The consumer is responsible for keeping track of its own position (the offset) in the log. Multiple consumers can read the same message at different times.
- Kafka uses topics to send and receive messages. Each topic can have multiple partitions. Each partition contains a subset of the messages for a specific topic. 
- Kafka ensures that messages are ordered within a partition, but doesn't guarantee ordering across different partitions in the same topic. If you need global ordering, you must use a single partition (which kills performance)
- Partitions allow Kafka to scale horizontally. This means that each partition can be processed in parallel by multiple machines, making Kafka capable of handling large-scale data streams.
- If there are N partitions for a topic and N consumers in the group, each consumer will be assigned to one partition, and each partition will be read by only one consumer.
- If we have 10 partitions and 15 consumers in one group then 5 consumers will sit idle. Kafka allows only one consumer per partition within a group to ensure ordering.
- We can increase partitions on the fly but can't decrease them without deleting and recreating the topic.
- Kafka follows a Leader-Follower model for partition replication. The leader broker is responsible for handling both read and write requests for a partition. Follower brokers replicate data from the leader. If the leader broker crashes, one of the follower brokers is elected to become the new leader, ensuring high availability and fault tolerance. Followers do not handle client requests directly, they simply replicate data from the leader.
- Kafka is an append-only log. To simulate it, you must create different topics (e.g., priority-high, priority-low) and have the consumer check the high-priority topic first.
- When the producer sends data faster than the consumer can process. Kafka handles this naturally because it's a "pull" model—the consumer just slows down.

Real-World Use Cases
- LinkedIn: Originally developed Kafka to handle activity streams and operational metrics.
- Netflix: Uses Kafka for real-time monitoring and recommendations.
- Uber: Processes geolocation and trip events in real time.
- Airbnb: Streams logs and analytics data for monitoring.

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

- Poll() is the Kafka client’s way of fetching events and keeping the consumer alive. It’s single-threaded, must be called regularly, and is the entry point for messages, errors, and rebalance notifications. In Go, We wrap it in a loop and handle each event type appropriately.

### Reliability & Data Integrity (Producer):
Reliability in Kafka means making sure that once We send a message, it doesn’t get lost or duplicated. The producer achieves this by waiting for acknowledgements (acks=all) from all replicas in the cluster. In Go, We configure this in the producer settings. We also enable idempotence, which ensures retries don’t create duplicates.

When We set acks=all, Kafka waits until the leader and all in-sync replicas (ISR) have written the message before confirming. This guarantees that even if the leader dies immediately after acknowledging, another replica already has the data and can take over.


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
