#  Go connection snippets

# Redis

```go
package main

import (
    "fmt"
    "github.com/redis/go-redis/v9"
    "context"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379", // host:port
    })

    err := rdb.Set(ctx, "key", "value", 0).Err()
    if err != nil {
        panic(err)
    }

    val, _ := rdb.Get(ctx, "key").Result()
    fmt.Println("Redis value:", val)
}

```
# Kafka
```go
package main

import (
    "fmt"
    "github.com/segmentio/kafka-go"
)

func main() {
    r := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"}, // host:port
        Topic:   "test-topic",
        GroupID: "group-1",
    })

    msg, err := r.ReadMessage(nil)
    if err != nil {
        panic(err)
    }
    fmt.Println("Kafka message:", string(msg.Value))
}

```

# RabbitMQ
```go
package main

import (
    "fmt"
    "log"
    "github.com/streadway/amqp"
)

func main() {
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/") // host:port
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ch, _ := conn.Channel()
    q, _ := ch.QueueDeclare("test-queue", false, false, false, false, nil)
    fmt.Println("RabbitMQ queue:", q.Name)
}

```

# gRPC

```go
package main

import (
    "log"
    "google.golang.org/grpc"
)

func main() {
    conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure()) // host:port
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    log.Println("Connected to gRPC server")
}

```

# PostgreSQL
```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/lib/pq"
)

func main() {
    connStr := "postgres://user:password@localhost:5432/mydb?sslmode=disable"
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    fmt.Println("Connected to PostgreSQL")
}

```

# MongoDB

```go
package main

import (
    "context"
    "fmt"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
    client, err := mongo.Connect(context.Background(),
        options.Client().ApplyURI("mongodb://localhost:27017")) // host:port
    if err != nil {
        panic(err)
    }
    defer client.Disconnect(context.Background())

    fmt.Println("Connected to MongoDB")
}

```

# AWS

Driver: Always use github.com/aws/aws-sdk-go-v2 (latest official AWS SDK for Go).

Auth: By default, credentials are loaded from ~/.aws/credentials or environment variables.

Region: Must be set explicitly (us-east-1, ap-south-1, etc.).

Connection string format: Each service uses serviceName.NewFromConfig(cfg).

## SETUP Common For ALL

```go

import (
    "context"
    "fmt"
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
    "github.com/aws/aws-sdk-go-v2/service/sqs"
    "github.com/aws/aws-sdk-go-v2/service/sns"
    "github.com/aws/aws-sdk-go-v2/service/lambda"
)

func loadConfig() aws.Config {
    cfg, err := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1")) // set your region
    if err != nil {
        panic(err)
    }
    return cfg
}

```

## Amazon S3 (Storage)

```go
func s3Example() {
    cfg := loadConfig()
    client := s3.NewFromConfig(cfg)

    // List buckets
    result, err := client.ListBuckets(context.TODO(), &s3.ListBucketsInput{})
    if err != nil {
        panic(err)
    }
    for _, b := range result.Buckets {
        fmt.Println("Bucket:", *b.Name)
    }
}

```

## Amazon SQS (Queue)
```go
func sqsExample() {
    cfg := loadConfig()
    client := sqs.NewFromConfig(cfg)

    // Send message
    _, err := client.SendMessage(context.TODO(), &sqs.SendMessageInput{
        QueueUrl:    aws.String("https://sqs.us-east-1.amazonaws.com/123456789012/myqueue"),
        MessageBody: aws.String("Hello from Go!"),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println("Message sent to SQS")
}

```

## Amazon SNS (Notification)

```go
func snsExample() {
    cfg := loadConfig()
    client := sns.NewFromConfig(cfg)

    // Publish message
    _, err := client.Publish(context.TODO(), &sns.PublishInput{
        TopicArn: aws.String("arn:aws:sns:us-east-1:123456789012:mytopic"),
        Message:  aws.String("Hello from Go via SNS!"),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println("Message published to SNS")
}

```

## AWS Lambda (Serverless)

```go

func lambdaExample() {
    cfg := loadConfig()
    client := lambda.NewFromConfig(cfg)

    // Invoke Lambda function
    resp, err := client.Invoke(context.TODO(), &lambda.InvokeInput{
        FunctionName: aws.String("myLambdaFunction"),
        Payload:      []byte(`{"key":"value"}`),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println("Lambda response:", string(resp.Payload))
}

```
