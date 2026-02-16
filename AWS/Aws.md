# AWS

## Most Used AWS Services and Their Use Cases

### Amazon EC2
AWS EC2, or Elastic Compute Cloud, provides scalable virtual servers in the cloud. It is one of the most fundamental services because it gives organizations complete control over the operating system, networking, and storage. EC2 is used to host web applications, run enterprise workloads, and perform high-performance computing tasks. It supports features like Auto Scaling and Elastic Load Balancing, which allow applications to handle variable traffic efficiently.

### Amazon EBS
Amazon EBS is a block-level storage service designed for use with Amazon EC2 instances. It's like a virtual hard drive that can be attached to EC2 instances to store data, operating system files, applications, and more. Unlike traditional hard drives, EBS volumes are persistent, meaning data remains intact even when the EC2 instance is stopped or terminated (unless explicitly deleted).

### Amazon S3
Amazon S3 is a highly scalable object storage service that allows you to store and retrieve an unlimited amount of data. It’s perfect for a wide range of use cases, from backup storage to big data analytics and media hosting. You can organize your data using buckets (containers for objects), and each object is uniquely identified by a key within a bucket. I’ve used EBS volumes to store operating system files, databases, and application data that requires fast access and performance. For example, if you're running a database on EC2, you’d want to store that data on EBS. On the other hand, I’ve used S3 for storing static assets, backups, and logs because of its durability and cost-effectiveness for large-scale storage. I also use EBS snapshots to back up EC2 instance volumes and then store those backups in S3 to combine both storage types for different use cases.

Amazon S3 provides multiple storage classes, each tailored to different access patterns and cost requirements. Let’s look at three of the most common ones in a paragraph style explanation:

The S3 Standard storage class is designed for frequently accessed data. It offers high durability (99.999999999%) and availability, making it ideal for dynamic websites, mobile applications, and content distribution. Because it prioritizes speed and reliability, it comes with higher storage costs compared to other classes, but retrieval is always instant.

For data that is not accessed often but still needs to be quickly available when required, S3 Standard-Infrequent Access (IA) is a better fit. It provides the same durability as Standard but at a lower storage cost. The trade-off is that retrieval incurs a fee, making it suitable for backups, disaster recovery files, or datasets that are rarely queried but must remain accessible.

Finally, S3 Glacier is optimized for archival storage. It is the most cost-effective option for long-term data retention, such as compliance records or historical archives. While storage costs are extremely low, retrieval times can range from minutes to hours depending on the retrieval option chosen. This makes Glacier perfect for data you rarely need but must preserve securely for years.

### Amazon RDS
Amazon RDS, or Relational Database Service, is a managed database service that supports engines such as MySQL, PostgreSQL, Oracle, and SQL Server. RDS is valuable because it automates backups, patching, and scaling, which reduces operational overhead. It is commonly used in transactional applications like e-commerce platforms, ERP systems, and financial applications. RDS supports Multi-AZ deployments for high availability and read replicas for scaling read-heavy workloads. By offloading database management tasks, RDS allows developers to focus on application logic rather than infrastructure.

### AWS Lambda
AWS Lambda is a serverless compute service that runs code in response to events without requiring server management. It is widely adopted because it enables event-driven architectures and scales automatically. Lambda is used for automation, lightweight APIs, and IoT data processing. For example, when a file is uploaded to S3, Lambda can trigger a function to process it instantly. Lambda integrates with services like DynamoDB, API Gateway, and CloudWatch, making it central to serverless workflows. It is chosen when organizations want to reduce operational overhead and pay only for actual execution time.

### Amazon DynamoDB
Amazon DynamoDB is a fully managed NoSQL database that delivers single-digit millisecond latency at scale. It is designed for applications that require high throughput and predictable performance. DynamoDB is commonly used in gaming, IoT, and real-time analytics. Features like global tables, DynamoDB Streams, and on-demand capacity make it highly scalable and flexible. Organizations choose DynamoDB when relational models are not required but speed and scalability are critical.

### Amazon CloudFront
Amazon CloudFront is a content delivery network that distributes content globally with low latency. It improves user experience by caching content close to end users. CloudFront is widely used for streaming video, serving static files, and accelerating APIs. It integrates tightly with S3 and EC2, and supports advanced security features like AWS Shield and Web Application Firewall (WAF) to protect against DDoS attacks. Organizations choose CloudFront when they need to deliver content quickly and securely to a global audience.

### AWS IAM
AWS Identity and Access Management, or IAM, is the core service for managing access to AWS resources. It allows organizations to create users, groups, and roles, and define fine-grained permissions through policies. 

### Amazon VPC
Amazon Virtual Private Cloud, or VPC, provides isolated networking environments in AWS. It allows organizations to design their own network topology, including subnets, routing tables, NAT gateways, and VPN connections. VPC is critical for separating workloads, controlling traffic flow, and connecting on-premises environments securely. It is the foundation for hybrid cloud setups and ensures that applications run in a secure and controlled environment. Organizations choose VPC when they need flexibility in networking and strong isolation between workloads.

### Amazon CloudWatch
Amazon CloudWatch is the monitoring and observability service for AWS resources and applications. It collects metrics, logs, and events, and provides dashboards and alarms. CloudWatch is used to track performance, detect anomalies, and trigger automated actions. For example, if EC2 instances show unusual CPU spikes, CloudWatch can trigger auto scaling. It integrates with almost every AWS service, making it essential for maintaining visibility and reliability in production systems. Organizations rely on CloudWatch to ensure that their applications remain healthy and responsive.

### Amazon EKS
Amazon Elastic Kubernetes Service, or EKS, is a managed Kubernetes service that simplifies container orchestration. It is used for running microservices architectures, where scalability and resilience are critical. EKS handles the complexity of Kubernetes management, including upgrades and scaling, while integrating with IAM, VPC, and CloudWatch for security and observability. Organizations choose EKS when they want to run containerized workloads without managing Kubernetes manually.

### IAM vs Security Groups
AWS Identity and Access Management (IAM) and Security Groups (SG) are both security mechanisms, but they operate at different layers. IAM is about *who* can access AWS resources and *what actions* they can perform. It manages users, roles, and policies, enforcing the principle of least privilege. For example, IAM can allow a developer to launch EC2 instances but prevent them from deleting S3 buckets.   I use IAM to define who can access what, and under what conditions. For example, I create IAM users for individual team members, groups for roles like DevOps or Developers, and attach policies that define their permissions. I use roles for granting temporary permissions — such as allowing EC2 instances to access S3 buckets, or enabling Lambda to call DynamoDB.

Security Groups, on the other hand, are virtual firewalls that control *network traffic* to and from EC2 instances. They define rules for inbound and outbound traffic based on IP addresses, ports, and protocols. For example, a security group might allow HTTP traffic on port 80 from the internet but restrict SSH access to a specific IP range.    For example, if I allow incoming traffic on port 80 for HTTP, the outgoing response is automatically allowed — that's what we mean by stateful. I typically use security groups to restrict access to only necessary IP addresses or other AWS services. For instance, I might open port 22 only to my corporate IP for SSH, or restrict a database instance to accept traffic only from the web server’s security group, not from the public internet.

In short, IAM secures access at the identity and API level, while Security Groups secure access at the network level. Together, they form complementary layers of defense in AWS architectures.

---

### Region vs Availability Zones

AWS Regions and Availability Zones (AZs) are fundamental to AWS’s global infrastructure. A Region is a physical geographic area, such as `us-east-1` in Virginia or `ap-south-1` in Mumbai. Each Region is completely independent, with its own set of services and compliance standards. Organizations choose Regions based on proximity to users, regulatory requirements, or disaster recovery strategies.  

Within each Region, there are multiple Availability Zones. An Availability Zone is essentially a distinct data center with independent power, networking, and cooling. Availability Zones are connected with low-latency links, which allows applications to be architected for high availability. For example, deploying EC2 instances across multiple Availability Zones ensures that if one data center fails, the workload can continue running in another.  

The key difference is that Regions provide geographic separation, while Availability Zones provide fault tolerance within a Region. 

---

## AWS ASG vs ELB: Detailed Comparison

An **Auto Scaling Group (ASG)** and an **Elastic Load Balancer (ELB)** are two core AWS services that often work together, but they serve different purposes.  

An **ASG** is responsible for scaling compute resources. It monitors metrics such as CPU utilization or request counts, and automatically increases or decreases the number of EC2 instances based on defined policies. For example, during peak traffic hours, an ASG might launch additional EC2 instances to handle the load, and then terminate them when demand drops. This ensures cost efficiency and elasticity. ASGs also improve resilience by automatically replacing unhealthy instances, maintaining the desired capacity at all times.

An **ELB**, on the other hand, is responsible for distributing traffic. It acts as a single entry point for clients and balances incoming requests across multiple EC2 instances in one or more Availability Zones. ELBs improve application availability by ensuring that traffic is not sent to unhealthy instances. They also support advanced features like SSL termination, sticky sessions, and cross-zone load balancing. ELBs come in different types: Application Load Balancer (ALB) for HTTP/HTTPS traffic, Network Load Balancer (NLB) for ultra-low latency TCP/UDP traffic, and Gateway Load Balancer (GLB) for third-party appliances.

The key difference is that **ASG manages the number of instances**, while **ELB manages how traffic is distributed among those instances**. When combined, they provide both scalability and high availability. For example, an ASG can scale out EC2 instances during a traffic spike, and the ELB will automatically start routing requests to the new instances without manual intervention. This partnership is what makes AWS architectures resilient and cost-effective.

---

## Comparison Table

| **Feature** | **Auto Scaling Group (ASG)** | **Elastic Load Balancer (ELB)** |
|-------------|-------------------------------|---------------------------------|
| **Purpose** | **Automatically adjusts EC2 capacity** | **Distributes traffic across instances** |
| **Focus** | **Scalability and elasticity** | **High availability and fault tolerance** |
| **Key Function** | **Launches/terminates EC2 instances** | **Routes requests to healthy targets** |
| **Trigger** | **Metrics like CPU, requests, health checks** | **Incoming client traffic** |
| **Resilience** | **Replaces unhealthy instances automatically** | **Stops sending traffic to unhealthy instances** |
| **Types** | **Scaling policies: target tracking, step, scheduled** | **ALB, NLB, GLB** |
| **Integration** | **Works with ELB for traffic distribution** | **Works with ASG for scaling capacity** |

---
# AWS Security

### IAM vs Security Groups
AWS Identity and Access Management (IAM) and Security Groups (SG) are both security mechanisms, but they operate at different layers. IAM is about *who* can access AWS resources and *what actions* they can perform. It manages users, roles, and policies, enforcing the principle of least privilege. For example, IAM can allow a developer to launch EC2 instances but prevent them from deleting S3 buckets.  

Security Groups, on the other hand, are virtual firewalls that control *network traffic* to and from EC2 instances. They define rules for inbound and outbound traffic based on IP addresses, ports, and protocols. For example, a security group might allow HTTP traffic on port 80 from the internet but restrict SSH access to a specific IP range.  

In short, IAM secures access at the identity and API level, while Security Groups secure access at the network level. Together, they form complementary layers of defense in AWS architectures.

### AWS Security Mechanisms: SSH/SCP vs Service Access (S3, SQS)

AWS uses different security models depending on how you connect.  

When I connect to an EC2 instance using SSH or SCP, the security is enforced at the network and identity level. I authenticate using an SSH key pair, where the private key stays with me and the public key is registered with AWS. The EC2 instance itself is protected by Security Groups, which act as virtual firewalls. For example, I would configure the Security Group to allow inbound traffic on port 22 only from my office IP range or through a VPN.  
#### Step 1: Generate SSH Key Pair Locally
On your local machine (Linux/Mac):
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/my-ec2-key
```

Keep the private key safe; only upload the public key. 
```bash
aws ec2 import-key-pair --key-name my-ec2-key --public-key-material file://~/.ssh/my-ec2-key.pub
```
This command does not mention any EC2 instance name or ID. That’s because this step is not tied to a specific instance — it’s about registering the key pair with AWS.

Later, when you launch an EC2 instance, you specify the key pair name:
```bash
aws ec2 run-instances --image-id ami-12345678 --instance-type t2.micro --key-name my-ec2-key ...

```
### How it looks in the AWS Dashboard
    Go to EC2 → Instances → Launch Instance.

    In the “Key pair (login)” section, you’ll see a dropdown list of all key pairs registered in your account/region.

    Select the key pair name you imported (e.g., my-ec2-key).

    AWS will then inject the public key into the instance’s ~/.ssh/authorized_keys file.

    On your local machine, you use the private key (~/.ssh/my-ec2-key) to connect via SSH.

```bash
ssh -i ~/.ssh/my-ec2-key ubuntu@<EC2-Public-IP>

```
When my application, such as a Go binary, needs to upload data to S3 or send messages to SQS, the security model is different. Instead of relying on SSH keys, AWS uses IAM roles and policies to control access. I would assign an IAM role to the EC2 instance or container running the Go code, and that role would grant temporary credentials through the AWS Security Token Service. These credentials are short‑lived and scoped to specific actions, such as `s3:PutObject` for uploading files or `sqs:SendMessage` for sending messages. 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
    }
  ]
}
```
### When you launch a new EC2 instance from the AWS Dashboard:

    Go to EC2 → Instances → Launch Instance.

    In the launch wizard, there’s a section called “IAM role”.

    From the dropdown, you select the IAM role you created (for example, one that allows s3:PutObject or sqs:SendMessage).

    AWS automatically attaches that role to the instance.

    Inside the instance, the AWS SDK (like your Go code) can fetch temporary credentials via the Instance Metadata Service (IMDS), which are issued by the AWS Security Token Service (STS).


In summary, EC2 access through SSH or SCP relies on key pairs and Security Groups to secure host‑level connections, while service access to S3 or SQS relies on IAM roles, scoped policies, and temporary credentials to secure application‑level interactions. This distinction highlights how AWS separates infrastructure security from service security, and it shows that I can design architectures that protect both human users and applications effectively.



## SSL Termination in AWS

SSL termination refers to the point where encrypted HTTPS traffic is decrypted during its journey from the client to the server. In AWS, this usually happens at the **Elastic Load Balancer (ELB)**. The load balancer receives encrypted traffic from clients, uses an SSL/TLS certificate to decrypt it, and then forwards the unencrypted traffic (HTTP) to backend EC2 instances.  

The main advantage of SSL termination is **performance and simplicity**. By offloading the encryption and decryption process to the load balancer, backend servers don’t need to handle the heavy CPU work of SSL/TLS handshakes. This reduces resource usage and allows certificates to be managed centrally at the load balancer instead of individually on each server.  

However, SSL termination means that traffic between the load balancer and backend servers is unencrypted. This is acceptable in many cases because AWS ensures that traffic inside a VPC is secure and isolated. But for workloads that require **end-to-end encryption** (such as financial or healthcare applications), AWS supports **SSL passthrough** or **re-encryption**, where the load balancer decrypts traffic and then re-encrypts it before sending it to backend servers.

## Example
- **With SSL termination:** Client → HTTPS → ELB (decrypted) → HTTP → EC2  
- **With end-to-end encryption:** Client → HTTPS → ELB (decrypted) → HTTPS → EC2
---
## Build Go Binaries
```bash
GOOS=linux GOARCH=amd64 go build -o user main.go
```

This command is telling the Go compiler to **cross‑compile** a program for a specific target operating system and architecture.

- **`GOOS=linux`**: Sets the target operating system to Linux. Even if you run the build on macOS or Windows, the resulting binary will be built for Linux.  
- **`GOARCH=amd64`**: Sets the target CPU architecture to 64‑bit x86 (AMD64/Intel64). This ensures the binary runs on 64‑bit Linux systems.  
- **`go build`**: Compiles the Go source code into a binary executable.  
- **`-o user`**: Names the output binary `user`. Without this flag, Go would default to naming it after the package.  
- **`main.go`**: Specifies the source file to compile. Since it contains the `main` package and `main()` function, it becomes an executable program.

In short, this command compiles `main.go` into a **Linux 64‑bit executable** named `user`. You can then copy that binary to a Linux server and run it directly, without needing Go installed on the target machine.

---

## PORTS
- 80: Http
- 443: Https
- 22: SSH

## SSH to EC2 instance
ssh -i my.pem ec2-user@124:23:21:12

## SCP to EC2 instance
scp -i my.pem user ec2-user@124:23:21:12

---
## SQS
`Amazon SQS Queue Types`
- Standard Queue handles massive volumes of messages.At-least-once delivery — duplicates may occur. No guaranteed order — messages may arrive out of sequence. Use Cases: Background jobs, real-time data processing, microservice communication.
- FIFO Queue (First-In-First-Out) Exactly-once delivery — no duplicates. Guaranteed order — messages are processed as sent.Message grouping — groups can be processed in parallel, maintaining order within each. Use Cases: Financial transactions, inventory updates, workflow orchestration.

Amazon SQS is a fully managed message queuing service that helps decouple and scale microservices, distributed systems, and serverless applications. It allows you to send, store, and receive messages between software components at any volume, without losing messages. SQS supports both standard queues (the order in which messages are received is not guaranteed. So, messages could be processed out of order, every message will be delivered at least once, but there might be duplicates. ) and FIFO queues (ensures that the messages are processed in the exact order they were sent, each message will be delivered exactly one time, with no duplicates).

## SNS
`Two types of topics to suit different messaging needs: `
- Standard Topic High throughput — supports millions of messages per second. At-least-once delivery — messages may be duplicated. No guaranteed order — messages can arrive out of sequence. Supports multiple protocols — SQS, Lambda, HTTP/S, SMS, email, mobile push. Use Cases: Real-time alerts, fan-out messaging, background processing.
- FIFO Topic (First-In-First-Out) Strict ordering — messages are delivered exactly as published. Exactly-once delivery — no duplicates. Limited subscriptions — only supports FIFO SQS queues. Lower throughput — up to 300 messages/sec or 10 MB/sec. Use Cases: Financial transactions, inventory updates, workflows needing order.

Amazon SNS is a fully managed pub/sub messaging service that allows you to send messages to multiple subscribers simultaneously. It supports multiple protocols, including SMS, email, HTTP(S), and even Lambda functions. SNS enables the broadcasting of messages to a wide range of endpoints, making it ideal for use cases like application alerts, event notifications, and push notifications."
---

## End-to-End AWS Audio Processing Pipeline in Go


I used S3 to store music tracks, metadata, and other assets (like album covers) uploaded. When a Label uploads a new song and artwork and metadata, the file gets stored in an S3 bucket. This also triggered events for further processing using Lambda.

A Lambda function would automatically get triggered to generate different versions of the track (e.g., high-quality Flac, low-quality mp3, etc.) and store the processed files back in S3. Lambda helps here because it is serverless, automatically scaling as needed without the overhead of managing servers.

When Lambda finishes processing the song, it places a message in an SQS queue, signaling that the song is ready for distribution. Another service picks up the message from the queue to update the song's metadata in the database.

When the music processing is completed, an SNS message is sent to the artist to notify them that their track is live and available.

## Flow
1. **Client uploads file** → HTTP server stores file in **S3 input bucket** using multipart upload.  
2. **Server sends SQS message** → contains filename.  
3. **Lambda consumes SQS message** → downloads file from S3, converts with **FFmpeg**, uploads results to **S3 output bucket**.  
4. **Lambda publishes SNS notification** → informs subscribers (e.g., email).  
5. **Client can also invoke Lambda directly** → for synchronous processing.

---

## 1. HTTP Upload Service (S3 + SQS)

In Go, the AWS SDK provides a `s3manager.Uploader` which automatically handles multipart uploads. I just pass it a file stream, and it decides whether to use a single PutObject or multipart upload depending on the file size. This makes the code simple while still being efficient.

```go
package main

import (
    "fmt"
    "net/http"
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3/s3manager"
    "github.com/aws/aws-sdk-go/service/sqs"
)

func uploadHandler(w http.ResponseWriter, r *http.Request) {
    file, header, err := r.FormFile("file")
    if err != nil {
        http.Error(w, "File upload error", 400)
        return
    }
    defer file.Close()

    sess := session.Must(session.NewSession(&aws.Config{Region: aws.String("us-east-1")}))
    uploader := s3manager.NewUploader(sess) // This returns a pointer to an s3manager.Uploader object that you can use. 

    // Multipart upload to S3
    _, err = uploader.Upload(&s3manager.UploadInput{
        Bucket: aws.String("input-bucket"),
        Key:    aws.String(header.Filename),
        Body:   file,
    })
    if err != nil {
        http.Error(w, "S3 upload failed", 500)
        return
    }

    // Send message to SQS
    sqsClient := sqs.New(sess)
    _, err = sqsClient.SendMessage(&sqs.SendMessageInput{
        QueueUrl:    aws.String("https://sqs.us-east-1.amazonaws.com/123456789012/my-queue"),
        MessageBody: aws.String(fmt.Sprintf(`{"filename":"%s"}`, header.Filename)),
    })
    if err != nil {
        http.Error(w, "SQS send failed", 500)
        return
    }

    w.Write([]byte("Upload successful"))
}

func main() {
    http.HandleFunc("/upload", uploadHandler)
    http.ListenAndServe(":8080", nil)
}
```
NewUploader sets up sensible defaults (like part size, concurrency, thresholds for multipart upload).

You can override those defaults by passing options:
```go
uploader := s3manager.NewUploader(sess, func(u *s3manager.Uploader) {
    u.PartSize = 10 * 1024 * 1024 // 10 MB per part
    u.Concurrency = 5             // upload 5 parts in parallel
})

```
Once you have the uploader, you call uploader.Upload(...) to perform the upload.

If the file is larger than 5 MB, the uploader automatically switches to multipart upload.

## 2. Lambda Processor (S3 + FFmpeg + SNS)

```go
package main

import (
    "context"
    "encoding/json"
    "os/exec"
    "strings"
    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3/s3manager"
    "github.com/aws/aws-sdk-go/service/sns"
)

type Message struct {
    Filename string `json:"filename"`
}

func handler(ctx context.Context, sqsEvent events.SQSEvent) error {
    sess := session.Must(session.NewSession(&aws.Config{Region: aws.String("us-east-1")}))
    downloader := s3manager.NewDownloader(sess)
    uploader := s3manager.NewUploader(sess)
    s3Client := s3manager.NewUploader(sess)

    for _, record := range sqsEvent.Records {
        var msg Message
        json.Unmarshal([]byte(record.Body), &msg)

        input := "/tmp/" + msg.Filename
        file, _ := os.Create(input)
        defer file.Close()

        // Download file from S3
        downloader.Download(file, &s3.GetObjectInput{
            Bucket: aws.String("input-bucket"),
            Key:    aws.String(msg.Filename),
        })

        // Convert using FFmpeg
        base := strings.TrimSuffix(msg.Filename, ".mp3")
        flac := "/tmp/" + base + ".flac"
        wav := "/tmp/" + base + ".wav"
        exec.Command("/opt/bin/ffmpeg", "-i", input, flac).Run()
        exec.Command("/opt/bin/ffmpeg", "-i", input, wav).Run()

        // Upload converted files
        uploadFile := func(path, key string) {
            f, _ := os.Open(path)
            defer f.Close()
            uploader.Upload(&s3manager.UploadInput{
                Bucket: aws.String("output-bucket"),
                Key:    aws.String(key),
                Body:   f,
            })
        }
        uploadFile(flac, base+".flac")
        uploadFile(wav, base+".wav")
    }

    // Notify via SNS
    snsClient := sns.New(sess)
    _, err := snsClient.Publish(&sns.PublishInput{
        TopicArn: aws.String("arn:aws:sns:us-east-1:123456789012:AudioNotify"),
        Message:  aws.String("Your audio files have been converted and uploaded."),
        Subject:  aws.String("Audio Conversion Complete"),
    })
    return err
}

func main() {
    lambda.Start(handler)
}

```
Want to trigger the Lambda through an HTTP request so need AWS API Gateway
Lambda doesn’t natively expose an HTTP endpoint. API Gateway acts as the front door, letting users or apps call your Lambda via standard HTTP methods (GET, POST, etc.).
The handler in an AWS Lambda function is essentially the entry point—the method that gets executed when your function is invoked.

To deploy this Lambda function, we need to: Build the Go executable and Create the deployment package: 
```bash
GOOS=linux GOARCH=amd64 go build -o main
zip function.zip main 
```


- Upload the function.zip file to Lambda Console. 

- Set role to allow the Lambda function to access S3. 

- Go to the IAM Console: Attach a policy with s3:GetObject permission to the Lambda role. 

- Go to the API Gateway Console and create a new API. Choose HTTP API (simpler) or REST API (more features). Create a route (e.g., /download). Set Lambda function as the integration type and choose the Lambda function you just created.

curl "https://xyz12345.execute-api.us-west-2.amazonaws.com/dev/download?file=my-file.txt"


## 3. Client Invoking Lambda Directly
```go
package main

import (
    "context"
    "encoding/json"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/lambda"
    "github.com/aws/aws-sdk-go-v2/service/lambda/types"
)

func callLambda(functionName string, payload interface{}) (string, error) {
    cfg, _ := config.LoadDefaultConfig(context.TODO())
    client := lambda.NewFromConfig(cfg)

    body, _ := json.Marshal(payload)
    input := &lambda.InvokeInput{
        FunctionName:   &functionName,
        Payload:        body,
        InvocationType: types.InvocationTypeRequestResponse,
    }

    result, err := client.Invoke(context.TODO(), input)
    if err != nil {
        return "", err
    }
    return string(result.Payload), nil
}

```

## 4. SNS Subscription Example

The `snsClient.Subscribe` code is **not part of your Lambda handler or upload service logic**. Subscriptions are usually created once during setup or deployment, not every time your application runs. Think of it as **infrastructure configuration** rather than runtime business logic.

## Where to Put the Subscription Code

### Infrastructure Setup Phase
- Run the SNS subscription code as a one‑time script (or via AWS CLI/Console/Terraform/CloudFormation).
- Example: a small Go program that you execute once to subscribe an email endpoint to your topic.
- After subscription, AWS automatically delivers notifications whenever your Lambda publishes to that topic.

### Separate Admin Tool
- Keep the subscription code in a separate Go file (e.g., `subscribe.go`).
- Run it manually or as part of deployment pipelines when you need to add new subscribers.
- This avoids cluttering your Lambda or upload service with configuration logic.

### Do NOT Put Inside Lambda Handler
- If you put `snsClient.Subscribe` inside your Lambda, it would try to re‑subscribe on every invocation.
- This is unnecessary and could cause duplicate subscriptions.
- Lambda should only **publish messages to SNS**, not manage subscriptions.


```go
package main

import (
    "fmt"
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/sns"
)

func main() {
    sess := session.Must(session.NewSession(&aws.Config{Region: aws.String("us-east-1")}))
    snsClient := sns.New(sess)

    result, err := snsClient.Subscribe(&sns.SubscribeInput{
        TopicArn: aws.String("arn:aws:sns:us-east-1:123456789012:AudioNotify"),
        Protocol: aws.String("email"),
        Endpoint: aws.String("user@example.com"),
    })
    if err != nil {
        fmt.Println("Subscription failed:", err)
        return
    }

    fmt.Println("Subscription ARN:", *result.SubscriptionArn)
}

```

```
                                ┌───────────────────────────────┐
                                │           Client              │
                                │   Uploads audio file (HTTP)   │
                                └───────────────┬───────────────┘
                                                │
                                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   HTTP Upload Service (Go)                                                     │
│   Function: uploadHandler()                                                    │
│   ──────────────────────────────────────────────────────────────────────────  │
│   - Creates AWS Session: createSession()                                       │
│       sess := session.Must(session.NewSession(&aws.Config{Region: "us-east-1"}))│
│   - Uploads file to S3: uploadFile()  
        uploader := s3manager.NewUploader(sess)                                          │
│       uploader.Upload(&s3manager.UploadInput{Bucket:"input-bucket", Key:...})  │
│   - Sends SQS message: sendMessage()            
        sqsClient := sqs.New(sess)                               │
│       sqsClient.SendMessage(&sqs.SendMessageInput{QueueUrl:..., MessageBody:...})│
│   Requirements: IAM role with S3 PutObject + SQS SendMessage                   │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   S3 Input Bucket                                                              │
│   - Stores uploaded audio file                                                 │
│   - Encrypted with SSE-KMS                                                     │
│   - Bucket policy restricts access to Upload Service + Lambda                  │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   SQS Queue                                                                    │
│   - Message: {"filename":"file.mp3"}                                           │
│   - Dead Letter Queue configured for failures                                  │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   Lambda Processor (Go)                                                        │
│   Function: handler()                                                          │
│   ──────────────────────────────────────────────────────────────────────────  │
│   - Creates AWS Session: createSession()                                       │
│   - Downloads file from S3: downloadFile()                                     │
│       downloader.Download(file, &s3.GetObjectInput{Bucket:"input-bucket", Key:...})│
│   - Converts via FFmpeg: convertAudio()                                        │
│       exec.Command("ffmpeg", "-i", input, output)                              │
│   - Uploads results to S3: uploadFile()                                        │
│       uploader.Upload(&s3manager.UploadInput{Bucket:"output-bucket", Key:...}) │
│   - Publishes SNS notification: publishNotification()  
        snsClient := sns.New(sess)                        │
│       snsClient.Publish(&sns.PublishInput{TopicArn:..., Message:...})          │
│   Requirements: IAM role with S3 GetObject + PutObject + SNS Publish           │
│   Monitoring: CloudWatch Logs + X-Ray traces                                   │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   S3 Output Bucket                                                             │
│   - Stores converted files (.flac, .wav, etc.)                                 │
│   - Lifecycle policy: archive to Glacier after 30 days                         │
│   - Versioning enabled                                                         │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   SNS Topic                                                                    │
│   - Publishes notification to subscribers                                      │
│   - Subscribers: Email, other Lambdas                                          │
│   - Function: publishNotification()                                            │
│   Requirements: IAM policy restricts topic to trusted publishers               │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
                                v
┌───────────────────────────────────────────────────────────────────────────────┐
│   Subscribers                                                                  │
│   - Receive notification email                                                 │
│   - Could trigger downstream workflows                                         │
└───────────────────────────────────────────────────────────────────────────────┘


Optional Direct Invocation:
───────────────────────────────────────────────────────────────────────────────
Client ──> API Gateway ──> Lambda (synchronous processing)
- Function: callLambda()
- API Gateway handles authentication (IAM, Cognito, JWT)
- Lambda returns processed result directly
───────────────────────────────────────────────────────────────────────────────


```
