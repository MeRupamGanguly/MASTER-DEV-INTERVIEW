# GRPC

- gRPC is a high-performance, open-source RPC framework developed by Google. At its core, it uses HTTP/2 as the transport protocol, which gives it several advantages: multiplexed streams, server push, and efficient connection management. It relies on Protocol Buffers (protobuf) as the interface definition language, which ensures strongly typed contracts and backward compatibility across services

- Protocol Buffers are a language-neutral, platform-neutral serialization mechanism developed by Google. It is used by gRPC to define service contracts and message formats. 

- gNMI is a protocol defined by OpenConfig that uses gRPC for communication between network clients and devices. : Provides a unified way to configure devices, retrieve operational state, and subscribe to telemetry data.  Relies on YANG models for structured, vendor-neutral representation of configuration and state. Supports multiple encodings like JSON, JSON_IETF, Protobuf, and ASCII.

- YANG → Protobuf: In network automation, YANG defines the model (e.g., interface configuration, BGP state). Protobuf encodes the actual data when transported via gRPC/gNMI.

- YANG and Protobuf solve different problems. YANG is about modeling—defining what data exists, its hierarchy, and constraints, especially in networking. Protobuf is about serialization—how that data is efficiently transmitted between systems. In practice, I’ve used YANG models to ensure vendor-neutral configuration and telemetry definitions, and then relied on gNMI with Protobuf encoding to actually stream that data across services. This separation of concerns—model vs. transport—makes systems both interoperable and performant.



- Unary RPC – Client sends one request, gets one response. Server streaming – Client sends one request, server sends a stream of responses. Client streaming – Client sends a stream of requests, server responds once. Bi-directional streaming – Both client and server send a stream of messages.

- gRPC uses HTTP/2 and binary serialization (Protocol Buffers) for high performance and efficiency, It supports bi-directional streaming.
- REST relies on HTTP/1.1 and text-based formats like JSON, offering simplicity and ease of use for web applications. REST is stateless and leverages standard HTTP methods, making it widely adopted and easy to debug. 


```proto
syntax="proto3";

package gogen;

option go_package = "/gogen";

message TickerRequest {
    string symbol = 1;
}

message TickerResponse {
    string symbol = 1;
    double ltp = 2;
    string timestamp = 3; 
}

service TickerStreamService{
    rpc TickerStream (stream TickerRequest)returns(stream TickerResponse);
}
```

option go_package = "github.com/MeRupamGanguly/rupamic" this line decides where generated file will store.

Steps to gen grpc-go Codes:
```bash
docker run -it golang:1.23.6 /bin/bash
apt update
apt install protobuf-compiler
go version
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
docker cp ticker/ e3a5b0647829:/ticker/ #New Terminal from Host
protoc --proto_path=/ticker/domain --go_out=/ticker/domain --go-grpc_out=/ticker/domain /ticker/domain/ticker.proto
```
Second Create Dockerfile for generating grpc-go codes:
```dockerfile
FROM golang:1.23.6
RUN apt update
COPY /ticker /ticker
RUN apt install protobuf-compiler -y
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
RUN go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
RUN mkdir -p /ticker/domain/gogen
RUN protoc --proto_path=/ticker/domain --go_out=/ticker/domain --go-grpc_out=/ticker/domain /ticker/domain/ticker.proto
```
Build from Dockerfile
```bash
docker build -t grpcgen -f dockerfile.ticker .
docker run -it grpcgen
docker cp 52509fb095ed:/ticker/github.com/MeRupamGanguly/rupamic/ .
```



Transport Layer Security, commonly called TLS, encrypts the communication channel between a gRPC client and server so that messages and credentials cannot be read or tampered with by third parties. Token based authentication, such as JSON Web Tokens, provides identity and authorization at the application layer by requiring the client to include a signed token with each request. Using TLS together with JWT gives you two complementary protections: TLS secures the channel and prevents eavesdropping, while JWT proves who the caller is and what they are allowed to do.

For TLS to work, the server must possess a private key and a corresponding public certificate that is either self‑signed or signed by a trusted Certificate Authority. The client needs the CA certificate that signed the server certificate so it can verify the server’s identity during the TLS handshake. When mutual TLS is required, the client also holds its own private key and certificate, and the server must be configured to trust the CA that issued the client certificate so it can verify the client’s identity.

To create a private key you can run `openssl genpkey -algorithm RSA -out server.key -aes256`, which generates an encrypted RSA private key file. To create a self‑signed certificate for testing, you can run `openssl req -new -x509 -key server.key -out server.crt -days 365` and answer the prompts; the Common Name field should match the server’s hostname. For production, generate a Certificate Signing Request with `openssl req -new -key server.key -out server.csr` and submit the CSR to a trusted CA; after the CA signs it you will receive a server.crt that clients will trust when they have the CA certificate installed.

Simple one‑way TLS is appropriate for public facing services where clients need to verify the server but do not need to present their own identity; in this mode the server presents server.crt and server.key, and the client verifies the server using ca.crt. Mutual TLS is recommended for internal service‑to‑service communication where both sides must authenticate each other; in mTLS the client presents client.crt and client.key and the server verifies the client using a trust store containing the client CA. Choose one‑way TLS for broad compatibility and lower operational overhead, and choose mTLS when you need stronger, certificate‑based identity guarantees between machines.


On the server side in Go you load the server certificate and private key into gRPC transport credentials using `credentials.NewServerTLSFromFile("server.crt", "server.key")` and then create the gRPC server with `grpc.NewServer(grpc.Creds(creds))` so that all incoming connections are encrypted and the server proves its identity. To enforce token based authentication, add a unary interceptor that extracts and validates the JWT from the request context and rejects calls that do not present a valid token; this keeps authorization logic separate from transport security. On the client side, load the CA certificate with `credentials.NewClientTLSFromFile("ca.crt", "")` so the client verifies the server certificate during the handshake, and if mTLS is required configure the client to present its own certificate and key as part of the TLS configuration.


Use CA‑signed certificates in production rather than self‑signed certificates so clients can trust the server without manual pinning, and avoid pinning the server certificate directly because certificate rotation will break clients. Prefer mutual TLS for internal microservice communication where you need strong, cryptographic identity for both sides, and combine mTLS with short‑lived JWTs for fine‑grained authorization. Keep your private keys secure, automate certificate renewal where possible, and use the latest gRPC client APIs and explicit credential objects rather than deprecated helpers so your code remains maintainable and secure.



```go
// Load server certificate and private key for TLS
creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
```
```go
// Create a gRPC server with TLS credentials
grpcServer := grpc.NewServer(grpc.Creds(creds))
```

```go
// Create a gRPC server with interceptor for JWT validation in main
	grpcServer := grpc.NewServer(
		grpc.UnaryInterceptor(func(
			ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
			// Perform authentication
			if _, err := authenticate(ctx); err != nil {
				return nil, grpc.Errorf(grpc.Code(err), err.Error())
			}
			// Call the handler to proceed with the request
			return handler(ctx, req)
		}),
	)
```

The client must also use the server's public certificate to verify the server’s identity and establish a secure connection.

ca.crt (Correct): This is the root certificate that signed the server's certificate. By providing this, the client can verify that any certificate the server presents is legitimate and trusted.

server.crt (Incorrect/Brittle): While using the server's public key directly (called "certificate pinning") can work, it is not the standard practice. If the server rotates its certificate, the client will immediately stop working unless you update the client code too.

```go
// 1. Load the CA certificate (the trust anchor)
creds, err := credentials.NewClientTLSFromFile("ca.crt", "")
// Create a gRPC client with TLS credentials
// // Deprecated
conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))

// NEW Way
conn, err := grpc.NewClient("localhost:50051", 
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
```
1. grpc.Dial is Deprecated
You are using grpc.Dial, which has been officially deprecated. The replacement is grpc.NewClient. Unlike Dial, NewClient is "lazy"—it doesn't perform I/O immediately, making your application startup faster and more resilient.

2. grpc.WithInsecure() is Deprecated
Using WithInsecure() has been replaced by the more explicit insecure.NewCredentials(). This clarifies that you are intentionally choosing an unencrypted transport.

---


```proto
syntax = "proto3";

package chat;

service Chat {

  rpc ServerStream (Message) returns (stream Message);


  rpc ClientStream (stream Message) returns (Message);


  rpc StreamChat (stream Message) returns (stream Message);
}


message Message {
  string sender = 1;
  string content = 2;
}

```
- The Chat service contains three RPC methods.
- The Message message type contains two fields: sender (to indicate who sent the message) and content (the message text).

To generate the Go code from this .proto file, use the following command:
```bash
protoc --go_out=. --go-grpc_out=. chat.proto

protoc --proto_path=/ticker/domain --go_out=/ticker/domain --go-grpc_out=/ticker/domain /ticker/domain/ticker.proto
```
```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "github.com/youruser/project/proto/chat" // Adjust to your module path
)

type server struct {
	pb.UnimplementedChatServer // Required for forward compatibility
}

// 1. Server-side streaming: Server sends multiple messages
func (s *server) ServerStream(req *pb.Message, stream pb.Chat_ServerStreamServer) error {
	for i := 0; i < 5; i++ {
		msg := &pb.Message{
			Sender:  "Server",
			Content: fmt.Sprintf("%s (Time: %s)", req.Content, time.Now().Format(time.Kitchen)),
		}
		if err := stream.Send(msg); err != nil {
			return err
		}
		time.Sleep(500 * time.Millisecond)
	}
	return nil
}

// 2. Client-side streaming: Client sends many, Server returns one
func (s *server) ClientStream(stream pb.Chat_ClientStreamServer) error {
	var count int
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			// Finish and send summary
			return stream.SendAndClose(&pb.Message{
				Sender:  "Server",
				Content: fmt.Sprintf("Processed %d client messages.", count),
			})
		}
		if err != nil {
			return err
		}
		count++
		log.Printf("Received from client: %s", msg.Content)
	}
}

// 3. Bidirectional streaming: Independent simultaneous chat
func (s *server) StreamChat(stream pb.Chat_StreamChatServer) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		// Logic: Echo back immediately
		if err := stream.Send(&pb.Message{Sender: "Server", Content: "Echo: " + msg.Content}); err != nil {
			return err
		}
	}
}

func main() {
	// --- SERVER START ---
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterChatServer(s, &server{})
	log.Println("Server listening on :50051")
	go func() {
		if err := s.Serve(lis); err != nil {
			log.Fatalf("failed to serve: %v", err)
		}
	}()

	// --- CLIENT START ---
	// grpc.NewClient is the 2026 standard. grpc.Dial is legacy/deprecated.
	conn, err := grpc.NewClient("localhost:50051", 
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	client := pb.NewChatClient(conn)

	// A. Client-side streaming demonstration
	runClientStream(client)

	// B. Server-side streaming demonstration
	runServerStream(client)

	// C. Bidirectional streaming demonstration
	runBidiStream(client)
}

// helper // The client sends one request (like an ID or a search query), and the server responds by "pushing" a sequence of multiple messages back to the client. The client loops until it sees io.EOF, which signals the server is done.
func runClientStream(c pb.ChatClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, _ := c.ClientStream(ctx)
	for i := 1; i <= 3; i++ {
		stream.Send(&pb.Message{Content: fmt.Sprintf("Batch msg %d", i)})
	}
	res, _ := stream.CloseAndRecv()
	log.Printf("ClientStream Result: %s", res.Content)
}

// helper // The client sends one request (like an ID or a search query), and the server responds by "pushing" a sequence of multiple messages back to the client. The client loops until it sees io.EOF, which signals the server is done.
func runServerStream(c pb.ChatClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, _ := c.ServerStream(ctx, &pb.Message{Content: "Initial Ping"})
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			break
		}
		log.Printf("ServerStream Item: %s", msg.Content)
	}
}

// helper // This is the most complex and powerful. It creates a "full-duplex" connection where the client and server can both send and receive messages at the same time, independently.
func runBidiStream(c pb.ChatClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, _ := c.StreamChat(ctx)
	waitc := make(chan struct{})

	// Receive goroutine
	go func() {
		for {
			msg, err := stream.Recv()
			if err == io.EOF {
				close(waitc)
				return
			}
			log.Printf("Bidi Received: %s", msg.Content)
		}
	}()

	// Send logic
	for i := 0; i < 3; i++ {
		stream.Send(&pb.Message{Content: fmt.Sprintf("Bidi ping %d", i)})
		time.Sleep(200 * time.Millisecond)
	}
	stream.CloseSend()
	<-waitc
}
```
Summary of Steps

Define a Future-Proof Server: Create a server struct that embeds pb.UnimplementedChatServer. This is now required to ensure your code doesn't break when the .proto file is updated with new methods.

Implement Streaming with io.EOF Logic: * ServerStream: Uses the stream's Send() method within a loop.

ClientStream: Uses a for loop with stream.Recv(), specifically checking for io.EOF to trigger the final SendAndClose().

StreamChat (Bidi): Implements a non-blocking loop where Recv() and Send() act independently, typically returning nil only when the client closes the stream.

Initialize with grpc.NewServer(): Set up the gRPC engine, register the service, and begin transport handling on the listener.

Establish a "Lazy" Connection: Instead of grpc.Dial, use grpc.NewClient paired with insecure.NewCredentials(). This creates a resilient connection that handles internal load balancing and doesn't block your app if the server is temporarily unreachable.

Manage Streams with Contexts: Every client call must now use a context.WithTimeout or context.WithCancel. This ensures that if a stream hangs or a network partition occurs, resources are cleaned up automatically.

Handle Concurrency in Bidirectional Calls: When using StreamChat, use a Goroutine for the Recv() loop on the client side. This prevents a "head-of-line blocking" scenario where the client can't send because it's stuck waiting to receive.

# POINTS
- gRPC uses status codes and metadata to handle errors. For example: grpc.StatusCode.INVALID_ARGUMENT, grpc.StatusCode.UNAUTHENTICATED

- A server-streaming RPC in gRPC allows the server to send multiple messages in response to a single client request. After the client sends a request, the server can stream back a series of responses until the stream is closed. The client can receive messages continuously during the RPC.

- If the client or server does not properly manage the streaming connection (e.g., by not reading or sending messages in the expected order), the stream may hang, time out, or fail. The connection could also be prematurely closed, causing communication failures. Proper error handling and stream management are necessary to ensure the stability of the streaming RPC.

- Clients can specify a deadline or timeout for each RPC call. If the server does not respond within that time, the call fails with a DEADLINE_EXCEEDED error.

- Synchronous (blocking): Client waits for a response. Asynchronous (non-blocking): Client continues execution and handles response via callbacks or futures.

- Use versioned package names in .proto files. Maintain backward compatibility by adding new fields with optional/default values.

- gRPC supports built-in client-side load balancing. Integrates with DNS-based or custom name resolvers.

- Kubernetes assigns a DNS name to each Service (e.g., my-service.default.svc.cluster.local). gRPC clients can connect to these DNS names. Load balancing is handled by Kubernetes using kube-proxy (usually Round Robin).

            You define a Deployment for our gRPC server.

            You expose it via a Service (usually ClusterIP or LoadBalancer).

            The gRPC client calls the server using the service name.

            grpc.Dial("user-service.default.svc.cluster.local:50051", ...)

-  Network hiccups, transient server errors, or load balancer issues can cause temporary failures—retries give the system a second (or third) chance to succeed without user impact.  gRPC retries can be implemented in two main ways: using client-side service configuration or manual retry logic via interceptors. With service config (supported in Go and Java), we define a retryPolicy in the service’s config JSON, specifying rules like max attempts, backoff, and retryable status codes—this is automatic and built into the gRPC client. On the other hand, interceptors allow us to manually code retry logic by wrapping client calls and reissuing them based on custom conditions. While service config is simpler and declarative, interceptors offer more control and flexibility. For basic retry needs, service config is recommended when available.
