# RabbitMQ V/S Kafka
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

To secure RabbitMQ or Kafka with TLS, the server always needs a private key and a certificate signed by a trusted CA. The client must have the CA certificate to verify the server. If mutual TLS is required, then the client also needs its own private key and certificate, and the server must trust that client certificate. In Kafka specifically, these are stored in keystores and truststores, while in RabbitMQ they are configured directly as PEM files.
    
      RabbitMQ TLS Keys and Certificates
            Server (RabbitMQ broker):
                Private key → used to prove the server’s identity.
                Server certificate → signed by a trusted CA, presented to clients.
                CA certificate (trust store) → used to validate client certificates if mutual TLS is enabled.
            
            Client (RabbitMQ consumer/producer):
                CA certificate → to verify the RabbitMQ server’s certificate.
                (Optional, for mTLS) Client private key + client certificate → presented to RabbitMQ if the server requires client authentication.
            
            Without client certificates, RabbitMQ only authenticates the server. With client certificates, both sides authenticate each other.
      
      Kafka TLS Keys and Certificates
            Server (Kafka broker):
               Private key → unique to each broker.
               Broker certificate → signed by a CA, stored in a keystore.
               Truststore (CA certificate) → used to validate client certificates if mTLS is enabled.
            
            Client (Kafka producer/consumer):
               Truststore (CA certificate) → to verify the broker’s certificate.
               (Optional, for mTLS) Client private key + client certificate → stored in a keystore, presented to Kafka broker if required.
            
            Kafka uses Java keystores and truststores (PKCS12 or JKS format). Each broker and client must have its own keystore with private key + certificate, and a truststore containing CA certificates.
          
RabbitMQ and Kafka Default Ports:
```go
import amqp "github.com/rabbitmq/amqp091-go"
conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
// 2. Open a channel
    ch, _ := conn.Channel()
````
```go
import "github.com/confluentinc/confluent-kafka-go/v2/kafka"
p, _ := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "client.id":         "go-producer",
        "acks":              "all",
    })
```

# KAFKA 
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
`bootstrap.servers` → The Kafka broker(s) your producer connects to. "localhost:9092" means it’s running locally. In production, you’d list multiple brokers for resilience.

`acks=all` → The producer waits until all in-sync replicas acknowledge the message. This guarantees durability (no data loss if the leader crashes).

`enable.idempotence=true` → Ensures exactly-once semantics for the producer. Prevents duplicates if retries happen. Critical for financial or transactional systems.

`compression.type=lz4` → Compresses messages before sending. LZ4 is fast and efficient, reducing network + disk usage, improving throughput.

`linger.ms=10` → Waits up to 10ms before sending a batch. This allows multiple messages to be grouped together, improving efficiency.

`batch.num.messages=1000` → Maximum number of messages per batch. Larger batches = better throughput, but more latency.

`message.timeout.ms=60000` → If a message cannot be delivered within 60 seconds, fail it. Prevents infinite retries and lets We handle errors quickly.

`bootstrap.servers `→ Same as producer: list of brokers to connect to.

`group.id=payments-service` → Identifies the consumer group. All consumers with the same group ID share partitions. Here, it’s named after the service.

`auto.offset.reset=earliest` → If no committed offset exists, start reading from the earliest available message. Useful for new consumers that want the full history.

`enable.auto.commit=false` → Disables automatic offset commits. We commit offsets manually after successful processing. This prevents “false progress” if a message fails.

`max.poll.interval.ms=300000 (5 minutes) `→ Maximum time between Poll() calls before Kafka considers the consumer dead. Setting it high allows long processing tasks without triggering a rebalance.

`fetch.min.bytes=1024 (1 KB)` → Broker waits until at least 1 KB of data is available before responding. Helps batch small messages together for efficiency.

`fetch.max.bytes=52428800 (50 MB)` → Maximum data per fetch request. Prevents a single consumer from being overwhelmed by huge batches.


## Security

SSL (TLS) → encrypts traffic between client and broker. SASL (Simple Authentication and Security Layer) → provides authentication (username/password, Kerberos, OAuth, etc.). Together, SASL/SSL ensures both secure communication and identity verification.

`security.protocol=SASL_SSL ` → This tells the Kafka client to use SASL authentication over an SSL/TLS encrypted connection. SSL ensures the traffic between client and broker is encrypted. SASL provides authentication (so the broker knows who We are). Together, this prevents eavesdropping and unauthorized access.

`sasl.mechanisms=PLAIN  `
Defines the SASL mechanism used for authentication.

`PLAIN` → Username/password in plain text (but sent securely inside SSL).

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

Backpressure happens when a consumer cannot keep up with the rate at which a producer sends data. In Kafka, consumers pull data rather than having it pushed to them, so if a consumer stops polling, the data simply waits in the broker. However, if the consumer stops polling for too long, Kafka assumes the consumer is dead and reassigns its partitions. To avoid this, Kafka provides a mechanism to pause and resume consumption. When a consumer calls Pause(), Kafka stops sending new data for that partition but continues to receive heartbeats, which lets Kafka know the consumer is still alive. This allows the consumer time to process its backlog without losing its place. Once the consumer is ready to continue, it calls Resume(), and Kafka resumes sending messages from where it left off.

```go
consumer.Pause([]kafka.TopicPartition{{Topic: &topic, Partition: 0}})
```

# RabbitMQ 
In RabbitMQ, queues should be declared in the consumer side. The publisher does not need to declare queues, but instead, it publishes messages to an exchange. The exchange is responsible for routing the messages to the appropriate queues based on the routing key and binding configuration.

Publisher will create Connection then from Connection it will create Channel then from channel it call ExchangeDeclare then it sends messages using Publish function of channel.

Consumer will create Connection then from Connection it will create Channel then from channel it call ExchangeDeclare then it call QueueDeclare then it Binds the Queue and Exchange by QueueBind function of Channel. Then call Consume function of channel.

In Topic Exchange Publisher sends messages to specific queues based on routing pattern specified by the routing key. The routing key can include wildcards (* for a single word and # for multiple words).

## Publisher

```go
func main() {
	// Establish a connection to RabbitMQ
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("Failed to connect: %s", err)
	}
	defer conn.Close() // Ensure connection is closed when the function exits
	// Create a channel to communicate with RabbitMQ
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Failed to create channel: %s", err)
	}
	defer ch.Close() // Ensure channel is closed when the function exits
	// Declare a topic exchange named "topic_logs"
	err = ch.ExchangeDeclare(
		"topic_logs",    // Exchange name
		"topic",     // Exchange type (topic exchange)
		true,        // Durable: The exchange will survive server restarts
		false,       // Auto-deleted: The exchange won't be deleted when no consumers are connected
		false,       // Internal: The exchange is not internal to RabbitMQ
		false,       // No-wait: Don't wait for confirmation when declaring the exchange
		nil,         // Additional arguments (none in this case)
	)
	if err != nil {
		log.Fatalf("Failed to declare exchange: %s", err)
	}
	// Publish some messages
	// Routing key that matches the pattern for consumers (in this case, it will match "animal.cat")
	messages := []struct {
		routingKey string
		body       string
	}{
		{"animal.cat", "Cat message"},
		{"animal.dog", "Dog message"},
		{"animal.bird", "Bird message"},
	}
	for _, msg := range messages {
		err = ch.Publish(
			exchange, msg.routingKey, false, false,
			amqp.Publishing{
				ContentType: "text/plain",
				Body:        []byte(msg.body),
			},
		)
		if err != nil {
			log.Fatalf("Failed to publish message: %s", err)
		}
	}
	fmt.Println("Message sent:", messages)
}
```
## Consumer
```go
// Consumer helper-function to handle message processing
func consumeMessages(ch *amqp.Channel, wg *sync.WaitGroup, queueName string, prefetchCount int) {
	defer wg.Done()
	// Set the prefetch count: This ensures the consumer will not receive more than `prefetchCount` messages at once
	err := ch.Qos(prefetchCount, 0, false) // The consumer will only receive 5 messages at once
	if err != nil {
		log.Fatalf("Failed to set Qos: %s", err)
	}
	// Start consuming messages from the queue
	msgs, err := ch.Consume(
		queueName, // Queue name
		"",        // Consumer tag (empty string means RabbitMQ will generate a tag)
		false,     // Auto-acknowledge messages (false means manual ack is required)
		false,     // Exclusive: This consumer is not exclusive to the connection
		false,     // No-local: Don't deliver messages published by the same connection
		false,     // No-wait: Don't wait for acknowledgment
		nil,       // Additional arguments (none in this case)
	)
	if err != nil {
		log.Fatalf("Failed to consume messages: %s", err)
	}
	// Process each message
	for msg := range msgs {
		// Simulate processing of the message (we can replace this with actual business logic)
		fmt.Printf("Processing message: %s\n", msg.Body)
		time.Sleep(1 * time.Second) // Simulate processing time (can be replaced with actual logic)
		// Acknowledge the message to let RabbitMQ know it's been processed
		msg.Ack(false)
	}
}
// CONSUMER
func main() {
	// Establish a connection to RabbitMQ
	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	if err != nil {
		log.Fatalf("Failed to connect: %s", err)
	}
	defer conn.Close() // Ensure connection is closed when the function exits
	// Create a channel to communicate with RabbitMQ
	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("Failed to create channel: %s", err)
	}
	defer ch.Close() // Ensure channel is closed when the function exits
	// Declare a topic exchange named "topic_logs"
	exchange := "topic_logs"
	err = ch.ExchangeDeclare(
		exchange,    // Exchange name
		"topic",     // Exchange type (topic exchange)
		true,        // Durable: The exchange will survive server restarts
		false,       // Auto-deleted: The exchange won't be deleted when no consumers are connected
		false,       // Internal: The exchange is not internal to RabbitMQ
		false,       // No-wait: Don't wait for confirmation when declaring the exchange
		nil,         // Additional arguments (none in this case)
	)
	if err != nil {
		log.Fatalf("Failed to declare exchange: %s", err)
	}
	// Declare a queue (anonymous, will be deleted after the consumer disconnects)
	queue, err := ch.QueueDeclare(
		"",    // Empty queue name (RabbitMQ generates a random queue name)
		false, // Durable: The queue won't survive server restarts
		true,  // Delete when unused: The queue will be deleted when no consumers are connected
		true,  // Exclusive: The queue is exclusive to this connection
		false, // No-wait: Don't wait for confirmation when declaring the queue
		nil,   // Additional arguments (none in this case). will use for dead letter exchange
	)
	if err != nil {
		log.Fatalf("Failed to declare queue: %s", err)
	}
	// Set prefetch count to 5 (each consumer will only receive 5 messages at a time)
	prefetchCount := 5
	// Bind the queue to the exchange with a routing key pattern "animal.*"
	routingKey := "animal.*"
	err = ch.QueueBind(
		queue.Name,   // Queue name
		routingKey,   // Routing key pattern
		exchange,     // Exchange to bind to
		false,        // No-wait: Don't wait for confirmation when binding the queue
		nil,          // Additional arguments (none in this case)
	)
	if err != nil {
		log.Fatalf("Failed to bind queue: %s", err)
	}

	// Use a wait group to wait for all consumers to finish processing
	var wg sync.WaitGroup
	numConsumers := 3 // Number of consumers to start concurrently
	for i := 0; i < numConsumers; i++ {
		wg.Add(1)
		go consumeMessages(ch, &wg, queue.Name, prefetchCount)
	}

	// Wait for all consumers to finish processing
	wg.Wait()
}
```

We are using topic exchange (topic_logs) with messages being published to it with routing keys such as animal.cat, animal.dog, animal.bird. The consumers are bound to receive messages that match the animal.* pattern, so they will get messages with keys like animal.cat, animal.dog, and so on.

### In Fanout Exchange Publisher sends messages to all Queues connected to Fanout Exchanges
```go
// Declare fanout exchange
ch.ExchangeDeclare("logs", "fanout", true, false, false, false, nil)

// Publish a message (routing key is ignored for fanout)
err = ch.Publish("logs", "", false, false, amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte("Fanout exchange message"),
})
```
```go
// Declare fanout exchange
ch.ExchangeDeclare("logs", "fanout", true, false, false, false, nil)

// Declare an anonymous queue
queue, err := ch.QueueDeclare("", false, true, true, false, nil)
if err != nil {
    log.Fatalf("Failed to declare queue: %s", err)
}

// Bind the queue to the exchange (routing key is ignored for fanout)
err = ch.QueueBind(queue.Name, "", "logs", false, nil)
````
### In Direct Exchange Publisher sends message to specific Queues based on a Exact match of Routing key.
```go
// Declare direct exchange
ch.ExchangeDeclare("direct_logs", "direct", true, false, false, false, nil)

// Publish a message with a routing key (e.g., "error")
err = ch.Publish("direct_logs", "error", false, false, amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte("Error in processing the task"),
})
```
```go
// Declare direct exchange
ch.ExchangeDeclare("direct_logs", "direct", true, false, false, false, nil)

// Declare an anonymous queue
queue, err := ch.QueueDeclare("", false, true, true, false, nil)
if err != nil {
    log.Fatalf("Failed to declare queue: %s", err)
}

// Bind the queue to the exchange with the routing key "error"
err = ch.QueueBind(queue.Name, "error", "direct_logs", false, nil)
```
### In Header Exchnage Publisher sends messages to all queues based on Message Headers rather than Routing Key.
```go
// Declare header exchange
ch.ExchangeDeclare("header_logs", "headers", true, false, false, false, nil)

// Define message headers
headers := amqp.Table{
    "X-Color": "Red",  // Set a header key-value pair
}

// Publish a message with headers
err = ch.Publish("header_logs", "", false, false, amqp.Publishing{
    Headers:     headers,
    ContentType: "text/plain",
    Body:        []byte("Header exchange message with color Red"),
})
```
```go
// Declare header exchange
ch.ExchangeDeclare("header_logs", "headers", true, false, false, false, nil)

// Declare an anonymous queue
queue, err := ch.QueueDeclare("", false, true, true, false, nil)
if err != nil {
    log.Fatalf("Failed to declare queue: %s", err)
}

// Bind the queue using header matching (e.g., "X-Color" equals "Red")
headers := amqp.Table{
    "X-Color": "Red",  // Match this header for routing
}
err = ch.QueueBind(queue.Name, "", "header_logs", false, headers)

```

`Durable Queue`: Survives server restarts, messages are not lost. In ch.QueueDeclare() function we have one boolean argument durable by which we can set it.

`Transient Queue`: Does not survive a server restart. In ch.QueueDeclare() function we have one boolean argument durable by which we can unset it and make it Transient.

`Exclusive Queue`: Only used by one consumer and deleted when that consumer disconnects. In ch.QueueDeclare() function we have one boolean argument exclusive by which we can set it.

`Auto-Delete Queue`: Deletes itself when no consumers are using it. In ch.QueueDeclare() function we have one boolean argument auto-delete by which we can set it.

`Dead Letter Queues`: Special queues for storing messages that can't be delivered (e.g., after too many retries).
```go
// Declare a dead letter exchange
	args := amqp.Table{
		"x-dead-letter-exchange": "dlx_exchange",
	}
	_, err = ch.QueueDeclare(
		"normal_queue",
		true,  // durable
		false, // delete when unused
		false, // exclusive
		false, // no-wait
		args,  // arguments for DLX
	)
```
`Automatic Acknowledgement`: The message is automatically confirmed once sent to the consumer. This is the default behavior. In ch.Consume() function we have one boolean argument auto-ack by which we can set it.

`Manual Acknowledgement`: The consumer confirms it has received and processed a message. In ch.Consume() function we have one boolean argument auto-ack by which we can unset it. In this case, the message is not considered acknowledged until we explicitly call the Ack method. msg.Ack(false).

`Negative Acknowledgement` (Nack): When a message is rejected by a consumer, RabbitMQ can retry or discard it. In this case, we can use Nack to reject the message, and optionally, we can requeue it for another attempt. msg.Nack(false, true) true to requeue the message

`Clustering`: Running RabbitMQ on multiple servers to spread the load and ensure it's always available.

`Mirrored Queues`: Copying queues to other servers in the cluster so that if one server fails, messages are still available.

`Federation`: Allowing RabbitMQ instances in different locations to share messages.

`Prefetch Count`: Limit how many messages a consumer can process at once to avoid overwhelming it.

`Concurrency`: Running many consumers in parallel to handle more messages.

`Flow Control`: Managing how messages are processed to avoid overloading RabbitMQ.


TLS encrypts the communication between clients and RabbitMQ servers, preventing eavesdropping or interception by third parties. Without TLS, messages and credentials sent over the network can be easily captured in plaintext, especially in a shared or insecure network environment (e.g., public Wi-Fi or unsecured VPNs).

TLS helps authenticate the server. When a client connects to RabbitMQ over a TLS-secured connection, it verifies the RabbitMQ server's certificate (using a CA certificate), ensuring the client is connecting to the right server and not an imposter.

We can configure RabbitMQ to request client certificates as part of the TLS handshake. This allows for mutual authentication, meaning both the server and the client verify each other's identity.

```go
// Load the CA cert
	caCert, err := os.ReadFile("/path/to/ca_certificate.pem")
	if err != nil {
		log.Fatalf("Failed to read CA cert: %v", err)
	}

	// Create a certificate pool from the CA cert
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(caCert)

	// Create the TLS config
	tlsConfig := &tls.Config{
		RootCAs: certPool,
	}

	// RabbitMQ URL
	rabbitmqURL := "amqps://user:password@our.rabbitmq.server:5671/"

	// Dial the connection with TLS
	conn, err := amqp.DialTLS(rabbitmqURL, tlsConfig)
	if err != nil {
		log.Fatalf("Failed to connect to RabbitMQ: %v", err)
	}
	defer conn.Close()
```

Client Certificates: If RabbitMQ requires a client certificate, we can load the certificate and private key into the tls.Config object.

```go
clientCert, err := tls.LoadX509KeyPair("/path/to/client_cert.pem", "/path/to/client_key.pem")
if err != nil {
    log.Fatalf("Failed to load client cert: %v", err)
}

tlsConfig := &tls.Config{
    RootCAs:      certPool,
    Certificates: []tls.Certificate{clientCert},
}
```
To secure RabbitMQ, I enable TLS so that all communication between clients and the server is encrypted, preventing eavesdropping. TLS also ensures server authentication by verifying the server’s certificate against a trusted CA. Additionally, RabbitMQ can be configured to request client certificates, which enables mutual authentication — both the server and the client verify each other’s identity. In practice, the steps are: load the CA certificate, create a certificate pool, build a TLS configuration, and then connect to RabbitMQ using a TLS‑enabled URL. If client certificates are required, I also load the client certificate and private key into the TLS configuration before establishing the connection.
- Load CA certificate → trust the server.
- Create certificate pool → store trusted CA.
- Build TLS config → attach CA pool.
- Dial RabbitMQ with TLS URL → secure connection.
- Add client certs to TLS config → both sides verify identity.
