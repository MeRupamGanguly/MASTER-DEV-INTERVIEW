
# Resilience patterns
Resilience patterns are techniques used to make systems reliable even when parts of them fail. In large distributed systems, failures are normal—servers crash, networks slow down, or services become unavailable. Instead of trying to prevent every failure, resilience patterns focus on handling them gracefully so the system continues to work.

One common pattern is the circuit breaker. Just like an electrical circuit breaker stops the flow of electricity when there is a problem, a software circuit breaker stops sending requests to a failing service. This prevents the system from wasting resources on repeated failures and gives the service time to recover.
This example shows a simple circuit breaker that opens after three failures, blocking further calls until reset.
```go
package main

import (
    "errors"
    "fmt"
)

type CircuitBreaker struct {
    failures int
    threshold int
    open bool
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.open {
        return errors.New("circuit is open, request blocked")
    }
    err := fn()
    if err != nil {
        cb.failures++
        if cb.failures >= cb.threshold {
            cb.open = true
        }
        return err
    }
    cb.failures = 0
    return nil
}

func main() {
    cb := &CircuitBreaker{threshold: 3}
    for i := 0; i < 5; i++ {
        err := cb.Call(func() error {
            return errors.New("service failed")
        })
        fmt.Println("Attempt:", i, "Error:", err)
    }
}

```

Another important pattern is retry with backoff. If a request fails, the system tries again, but instead of retrying immediately, it waits a little longer each time. This prevents overwhelming a struggling service with too many requests at once.
Here, the wait time doubles after each retry, preventing overload on the failing service.
```go
package main

import (
    "errors"
    "fmt"
    "time"
)

func retryWithBackoff(fn func() error, retries int) error {
    var err error
    for i := 0; i < retries; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        wait := time.Duration(1<<i) * time.Second
        fmt.Println("Retrying in", wait)
        time.Sleep(wait)
    }
    return err
}

func main() {
    err := retryWithBackoff(func() error {
        fmt.Println("Trying...")
        return errors.New("temporary failure")
    }, 3)
    fmt.Println("Final result:", err)
}

```
The bulkhead pattern is about isolating parts of the system so that one failure does not sink the entire application. For example, if one service is overloaded, bulkheads ensure that other services continue to run normally. This is similar to compartments in a ship that keep it afloat even if one section is damaged.
This example uses separate worker pools so if one pool is overloaded, others can still process jobs.
```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, jobs <-chan int) {
    for j := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, j)
        time.Sleep(time.Second)
    }
}

func main() {
    jobs := make(chan int, 5)

    // Two separate worker pools (bulkheads)
    go worker(1, jobs)
    go worker(2, jobs)

    for i := 1; i <= 5; i++ {
        jobs <- i
    }
    close(jobs)
    time.Sleep(3 * time.Second)
}

```
Rate limiting is also a resilience technique. It controls how many requests a service can handle at a time, protecting it from being overwhelmed by sudden spikes in traffic. Combined with graceful degradation, the system can still provide partial functionality instead of failing completely.
Here, requests are processed at a controlled pace, protecting the system from spikes.
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    limiter := time.Tick(500 * time.Millisecond) // 2 requests per second

    for i := 1; i <= 5; i++ {
        <-limiter
        fmt.Println("Processing request", i, "at", time.Now())
    }
}

```
Graceful shutdown ensures services stop safely without losing work.
```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    // Simulate server work
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Shutting down gracefully...")
                return
            default:
                fmt.Println("Working...")
                time.Sleep(time.Second)
            }
        }
    }()

    // Listen for Ctrl+C
    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt)
    <-c
    cancel()
    time.Sleep(2 * time.Second)
}

```
In Go, resilience patterns are often implemented using context.Context for timeouts and cancellations, middleware for retries, and libraries for circuit breakers. These tools allow developers to build services that remain responsive and stable even under heavy load or partial failures.

In short, resilience patterns are about designing systems that expect failure and continue to operate smoothly. They protect services from cascading problems, improve reliability, and ensure that users experience fewer disruptions even when parts of the system are struggling.
