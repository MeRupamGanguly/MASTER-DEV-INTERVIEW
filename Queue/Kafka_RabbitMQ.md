# RabbitMQ V/S Kafka

Kafka and RabbitMQ both support clustering, but their architectures reflect their design philosophies. Kafka clusters are built around brokers, topics, and partitions. Each topic is split into partitions distributed across brokers, with replication for fault tolerance. A leader handles reads and writes, while followers provide redundancy. This design allows Kafka to scale horizontally and handle millions of events per second, with the added benefit of replaying events since data is stored durably in a distributed commit log. On AWS, the easiest way to set this up is through Amazon MSK, which manages brokers, replication, and monitoring automatically.

Steps Managed Streaming for Apache Kafka:
- Open the Amazon MSK console.
- Click Create cluster → Choose Provisioned or Serverless.
- Select number of brokers (usually 3 across different AZs for HA).
- Configure VPC, subnets, security groups.
- Enable encryption, monitoring, and logging.
- Use AWS CLI or EC2 client to create topics, produce, and consume messages.

Advantages: Fully managed, automatic scaling, patching, monitoring, and integration with AWS services.


RabbitMQ clusters, on the other hand, are Erlang-based nodes where metadata like exchanges and bindings are replicated across the cluster. Queues typically live on a single node, but for high availability you configure mirrored or quorum queues, where one node acts as leader and others replicate messages. This ensures reliable delivery and failover, though scaling throughput is more limited compared to Kafka. On AWS, Amazon MQ provides a managed RabbitMQ option, handling clustering and failover for you. 

Steps Amazon MQ for RabbitMQ:
- Go to the Amazon MQ console.
- Choose Create broker → Select RabbitMQ engine.
- Pick an instance type (e.g., mq.m5.large) and deployment mode (single-instance or cluster).
- Configure VPC, subnets, and security groups for network access.
- Set up authentication (username/password or AWS Secrets Manager).
- Launch the broker → AWS provisions and manages RabbitMQ for you.

Advantages: Fully managed, automatic failover, monitoring via CloudWatch, easy scaling.

In short, Kafka clusters are optimized for scalability and event streaming, while RabbitMQ clusters are optimized for reliability and flexible routing in transactional workloads


## Ports and Connections:
KAFKA: 9093 RabbitMQ: 5672

```go
p, err := kafka.NewProducer(&kafka.ConfigMap{"bootstrap.servers": "my-secure-broker:9093",}
c, err := kafka.NewConsumer(&kafka.ConfigMap{"bootstrap.servers": "my-secure-broker:9093",}
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
```


# RabbitMQ

- RabbitMQ is an open-source message broker that implements the Advanced Message Queuing Protocol (AMQP), enabling reliable, asynchronous communication between distributed systems and microservices. 
- It’s not just a queue—it’s a robust middleware that decouples producers and consumers, ensuring scalability. 
- Producers and consumers don’t need to know about each other, RabbitMQ handles delivery of Messages that are stored in queues until consumed.  
- Producers don’t send messages directly to queues; they send them to exchanges, which route messages to queues based on rules. 
- It Supports acknowledgments, retries, and dead-letter queues to prevent message loss. Can handle millions of messages per second with clustering and federation. 
- RabbitMQ Built in Erlang, RabbitMQ efficiently manages thousands of concurrent connections.
- The broker manages the state of every message, tracking which consumer has acknowledged which task and then deleting the message once it is processed.
- Publisher will create Connection then from Connection it will create Channel then from channel it call ExchangeDeclare then it sends messages using Publish function of channel.
- Consumer will create Connection then from Connection it will create Channel then from channel it call ExchangeDeclare then it call QueueDeclare then it Binds the Queue and Exchange by QueueBind function of Channel. Then call Consume function of channel.
- RabbitMQ usually scales by "sharding" queues across nodes.

Real-World Use Cases
- eBay: Handles background tasks and decouples services for scalability.
- Instagram: Processes images/videos asynchronously after upload.
- Mozilla: Aggregates and routes logs in real time.
- BBC: Delivers real-time content updates across platforms.
- SoundCloud: Facilitates microservice communication via queues


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

## Security

- TLS encrypts the communication between clients and RabbitMQ servers, preventing eavesdropping or interception by third parties. Without TLS, messages and credentials sent over the network can be easily captured in plaintext, especially in a shared or insecure network environment (e.g., public Wi-Fi or unsecured VPNs).

- TLS helps authenticate the server. When a client connects to RabbitMQ over a TLS-secured connection, it verifies the RabbitMQ server's certificate (using a CA certificate), ensuring the client is connecting to the right server and not an imposter.

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
- Additionally, RabbitMQ can be configured to request client certificates, which enables mutual authentication — both the server and the client verify each other’s identity.

In practice, the steps are: load the CA certificate, create a certificate pool, build a TLS configuration, and then connect to RabbitMQ using a TLS‑enabled URL. If client certificates are required, I also load the client certificate and private key into the TLS configuration before establishing the connection.

- Load CA certificate → trust the server.
- Create certificate pool → store trusted CA.
- Build TLS config → attach CA pool.
- Dial RabbitMQ with TLS URL → secure connection.
- Add client certs to TLS config → both sides verify identity.


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

## TLS / Certs: Simple Checklist (Broker, Producer, Consumer, RabbitMQ server/client, AWS MQ, MSK)
A keystore holds your private key(s) and the certificate(s) that prove “you are you”; a truststore holds only public CA certificates that you trust to verify others. Keep private keys in a secret manager or HSM; keep trust anchors in read-only files that clients/brokers can load.

### What they are (concepts)
- **Keystore (what it contains):** a password‑protected file that **stores private keys and the corresponding certificate chain** (your cert + intermediate CAs). It proves identity when you must *present* a certificate (server or client). **Used for mTLS or server identity.**   
- **Truststore (what it contains):** a file that **stores only public CA certificates** (trusted root/intermediate certs). It is used to **verify** the certificate presented by the other side (broker or server). 

### Common formats
- **PKCS12 (.p12/.pfx)** — standardized, language-neutral, recommended for modern Java (default since Java 9).   
- **JKS (.jks)** — Java-specific legacy format (still supported).  
- **PEM (server.key + server.crt + ca.crt)** — common for non-Java systems (RabbitMQ, OpenSSL).  

### Who needs what (practical)
- **Kafka broker (self-managed):** **keystore** (server key+cert) to present identity; **truststore** if broker must validate client certs (mTLS). Configure `ssl.keystore.location` / `ssl.truststore.location`.   
- **Kafka producer/consumer:** **truststore** (always) to verify broker; **keystore** only if cluster requires client certs (mTLS).   
- **RabbitMQ server:** server cert + key (PEM); CA chain for client verification if mTLS enabled.   
- **RabbitMQ client:** CA cert to verify server; client cert/key if mTLS required. 

### Where files live (typical locations)
- **On Linux hosts:** `/etc/kafka/keystore.p12`, `/etc/kafka/truststore.jks`, `/etc/rabbitmq/certs/`  
- **In Kubernetes:** mounted from **Secrets** (preferably synced from Vault).  
- **Managed services (MSK / Amazon MQ):** download broker CA from console; clients import CA into truststore; client certs only if service configured for mTLS. 

### How to create / convert (examples)
- **Create PKCS12 keystore (key + cert):**
```bash
openssl pkcs12 -export -in cert.pem -inkey key.pem -certfile ca.pem -out keystore.p12 -name myalias
```
Create truststore (Java) from CA:

```bash
keytool -importcert -file ca.pem -alias myca -keystore truststore.jks
```
Convert JKS → PKCS12:

```bash
keytool -importkeystore -srckeystore keystore.jks -destkeystore keystore.p12 -deststoretype PKCS12
```
### Storage & security best practices (detailed)
Never commit private keys to source control. Store private keys and keystore passwords in a secret manager or HSM (HashiCorp Vault, AWS Secrets Manager, Cloud HSM). Use short‑lived certs and automate rotation. 

File permissions: owner-only (e.g., chmod 600), disk encryption at rest.

Kubernetes: use sealed/encrypted Secrets or external secret operator (sync from Vault).

Audit & rotate: log secret access and rotate certs/keys automatically.

### Quick Kafka / RabbitMQ config snippets
Kafka (server.properties):

```bash
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/etc/kafka/keystore.p12
ssl.keystore.password=*****
ssl.truststore.location=/etc/kafka/truststore.jks
ssl.truststore.password=*****
security.protocol=SSL
```
RabbitMQ (rabbitmq.conf):

```bash
listeners.ssl.default = 5671
ssl_options.cacertfile = /etc/rabbitmq/certs/ca.crt
ssl_options.certfile = /etc/rabbitmq/certs/server.crt
ssl_options.keyfile = /etc/rabbitmq/certs/server.key
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```
