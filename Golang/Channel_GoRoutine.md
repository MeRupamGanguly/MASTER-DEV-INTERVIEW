
# Channel

Channel is used to communicate via sending and receiving data and provide synchronisation between multiple gorountines. If channel have a value then execution blocked until reader reads from the channel. Channel can be buffered, allowing Go-Routines to send multiple values without blocking until the buffer is full.

```go
var ch chan int // Declares a channel variable but it’s nil until initialized via make. You cannot send or receive on a nil channel (blocks forever).
ch := make(chan int) // Creates a usable unbuffered channel. Send blocks until another goroutine receives, and vice versa.
ch := make(chan string, 5) // Creates a buffered channel with capacity 5. Allows up to 5 sends without a corresponding receive.

// Unlike maps or slices, Go does not support channel literals like ch := chan int{}. We must use make.
ch := new(chan int) // Creates a pointer to a nil channel. You still need to initialize it with make before use.
/*
 * What it is: Allocates memory for a chan int type, but does not initialize the channel.
 * Result: ch is of type *chan int (a pointer to a channel).
 * Default value: Points to a nil channel.
 */
```
```go
// new gives you a pointer to a chan int, but the channel itself is still nil.
ch := new(chan int) // type *chan int
for val := range *ch { //Dereferencing *ch is just a nil channel.
    fmt.Println(val) //Result: Same as var ch chan int → blocks forever.
}

```
```go
ch <- 42 // send value into channel
val := <-ch // receive value from channel
close(ch) // signal no more values will be sent

```

```go
for val := range ch { // Works until the channel is closed.
    fmt.Println(val)
}

```
for v := range ch only terminates when the channel is closed.

Trap: Forgetting to close(ch) → goroutine leaks, infinite loop. Only the sender should close a channel. tries to close from the receiver side → panic.

Can you close a channel multiple times?” → Answer: no, second close panics.

```go
select { // select lets you wait on multiple channels.
case msg := <-ch1:
    fmt.Println("Received:", msg)
case ch2 <- "data":
    fmt.Println("Sent data")
default: // forgets default case → goroutine blocks forever.
    fmt.Println("No communication")
}

```
How do you avoid blocking in select?” → Answer: add a default case or use timeouts with time.After

# Go-Routine

## Concurrency vs Parallelism
- Go-Routines are designed for concurrency, meaning multiple tasks can run using context switching. Threads are designed for parallelism, meaning multiple tasks can run simultaneously on multiple CPU cores.

- In context switching, imagine multiple tasks need to run at the same time. The Go-Runtime starts Task 1, and after some time, it pauses it and starts Task 2. After some time, it pauses Task 2 and can either start Task 3 or resume Task 1, and so on.

- Go-Routines have a dynamic stack size and are managed by the Go-Runtime. Threads have a fixed-size stack and are managed by the OS kernel. This dynamic growth of Stack ensures that memory is used efficiently. As a result, thousands or even millions of Go-Routines can be created and run concurrently with minimal memory overhead. This fixed size stack means that even if the thread doesn’t need all that memory, the space is reserved. As a result, threads consume more memory, which limits how many threads can run concurrently on a system.


## Go Runtime, Scheduler, and Logical Processor

### Go Runtime
- Go Runtime is the component that manages Scheduler, Garbage Collection, Concurrency.

    ### Go Scheduler
    - Go Scheduler is a component inside the Go runtime. The Go scheduler is responsible for deciding when and on which OS-Thread each Go-Routine runs.

        ### Logical Processor (P)
        - A Logical-processor is the Go scheduler’s unit that manages a queue of goroutines and maps them onto OS threads.


## Goroutine Lifecycle

When you start a goroutine using the go keyword, the Go runtime places it into a waiting list called the run‑queue. This run‑queue is managed by a logical processor. A logical processor is an internal unit of the Go scheduler that is responsible for keeping track of goroutines, their states, and coordinating their execution.

For example, on a machine with four cores, Go will typically create four logical processors. Each logical processor manages its own queue of goroutines and works with operating system threads to execute them.

A logical processor cannot execute goroutines by itself. It needs an operating system thread to actually run them. The Go scheduler attaches an operating system thread to each logical processor so that goroutines can be executed. At any given moment, a logical processor can only run one goroutine on its attached thread, even though its queue may contain many goroutines waiting.

If the thread becomes blocked (for example, waiting on a system call or I/O), the scheduler detaches that thread and assigns another available thread to the logical processor. This ensures that the logical processor continues to make progress and other goroutines in its queue are not stalled.

When a context switch happens (i.e., the runtime switches from one Go-Routine to another), Go saves the state of the current Go-Routine and loads the state of the next one. The OS-Thread then continues executing from where that new Go-Routine left off.

When a goroutine’s blocking condition is resolved (for example, data arrives on a channel or a syscall completes), the Go runtime places that goroutine back into a run‑queue.
It may go back into the same logical processor’s run‑queue where it was originally running.
Or, if that processor is busy, the scheduler can steal work and put it into another logical processor’s run‑queue

Because Go-Routines have very small stacks and are scheduled in user space, context switching between them is very fast and much cheaper than switching between OS-Threads.

```
                          ┌───────────────────────────────┐
                          │         Go Scheduler          │
                          │ Oversees processors & threads │
                          └───────────────┬───────────────┘
                                          │
        ┌─────────────────────────────────┼─────────────────────────────────┐
        │                                 │                                 │
        v                                 v                                 v
┌───────────────────┐          ┌───────────────────┐             ┌───────────────────┐
│ Logical Processor │          │ Logical Processor │             │ Logical Processor │
│        #1         │          │        #2         │             │        #3         │
│ Run-Queue: G1,G2  │          │ Run-Queue: G3,G4  │             │ Run-Queue: G5,G6  │
└─────────┬─────────┘          └─────────┬─────────┘             └─────────┬─────────┘
          │                              │                               │
          v                              v                               v
┌───────────────────┐          ┌───────────────────┐             ┌───────────────────┐
│ OS Thread T1      │          │ OS Thread T2      │             │ OS Thread T3      │
│ Running G1        │          │ Running G3        │             │ Running G5        │
└─────────┬─────────┘          └─────────┬─────────┘             └─────────┬─────────┘
          │                              │                               │
          v                              v                               v
   ┌───────────────┐             ┌───────────────┐              ┌───────────────┐
   │ G1 Preempted  │             │ G3 Blocked    │              │ G5 Yielded    │
   └───────┬───────┘             └───────┬───────┘              └───────┬───────┘
           │                             │                              │
           v                             v                              v
   Back to Run-Queue             ┌───────────────────────────────┐      Back to Run-Queue
                                 │ Scheduler detaches Thread T2  │
                                 │ Assigns free Thread T4 to LP2 │
                                 └─────────┬─────────────────────┘
                                           │
                                           v
                                 ┌───────────────────┐
                                 │ OS Thread T4      │
                                 │ Running G4        │
                                 └─────────┬─────────┘
                                           │
                                           v
                                 ┌───────────────────┐
                                 │ G3 Waiting State  │
                                 │ (blocked syscall) │
                                 └─────────┬─────────┘
                                           │
                                           v
                          Blocking condition resolved (e.g. data arrives)
                                           │
                                           v
                                 ┌───────────────────────────────┐
                                 │ G3 placed back into Run-Queue │
                                 │ Could go to LP2 or another LP │
                                 └───────────────────────────────┘

```

<img width="1408" height="768" alt="Image" src="https://github.com/user-attachments/assets/3aff1195-5b9f-4318-868e-5a179be2c4f8" />

## Controlling Execution

However, sometimes we might want to force Go to give other Go-Routines a chance to run even if the current Go-Routine is still doing something. This is where `runtime.Gosched()` comes in. When we call runtime.Gosched(), it tells Go to stop the current Go-Routine for a moment and allow other Go-Routines to run if they need to. It doesn't stop our Go-Routine permanently, but just gives other tasks a chance to run before our Go-Routine continues.

The number of Logical-Processors is controlled by GOMAXPROCS, and it determines how many Go-Routines can run in parallel (at the same time).

runtime.GOMAXPROCS(n): This function sets the maximum number of CPUs that Go will use for running Go-Routines. By default, Go will use all available CPUs. This function allows us to limit it to n CPUs. runtime.NumCPU(): This function returns the number of CPUs available on the current machine.



## Example: Yielding Goroutines

```go
func task1() {
    for i := 0; i < 5; i++ {
        fmt.Println("Task 1 - Iteration", i)
        if i == 2 {
            // Yield the CPU to other goroutines
            runtime.Gosched()
        }
        time.Sleep(time.Millisecond * 500)
    }
}

func task2() {
    for i := 0; i < 5; i++ {
        fmt.Println("Task 2 - Iteration", i)
        time.Sleep(time.Millisecond * 500)
    }
}
```
## Example: Parallel Fibonacci Calculation
```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// FibRecursive calculates Fibonacci numbers recursively
func FibRecursive(n int) int {
    if n <= 0 {
        return 0
    } else if n == 1 {
        return 1
    }
    return FibRecursive(n-1) + FibRecursive(n-2)
}

// Fibonacci calculation for each core
func calculateFibonacci(id int, cpun int, fibNum int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("CPU %d :- Go-Routine %d calculating Fibonacci(%d)\n", cpun, id, fibNum)
    result := FibRecursive(fibNum)
    fmt.Printf("CPU %d :- Go-Routine %d result: Fibonacci(%d) = %d\n", cpun, id, fibNum, result)
}

func main() {
    numCPU := runtime.NumCPU()
    fmt.Printf("Number of CPUs available: %d\n", numCPU)

    runtime.GOMAXPROCS(numCPU)

    var wg sync.WaitGroup

    for i := 0; i < numCPU; i++ {
        fibNum := 32 + i
        wg.Add(1)
        go calculateFibonacci(i+1, i, fibNum, &wg)
    }

    wg.Wait()
    time.Sleep(5 * time.Second)
    fmt.Println("DONE---------")
}

```
## Closure in golang
A closure in Go is a function that captures variables from its surrounding scope and “remembers” them even after that scope has finished executing. This allows the function to access and modify those variables later. Each closure maintains its own copy of the captured variables, so multiple closures can operate independently.

```go
func adder() func(int) int {
    sum := 0
    return func(x int) int {
        sum += x
        return sum
    }
}

func main() {
    posAdder := adder()
    negAdder := adder()

    fmt.Println(posAdder(1))  // 1
    fmt.Println(posAdder(2))  // 3
    fmt.Println(negAdder(-2)) // -2
    fmt.Println(negAdder(-3)) // -5
}

```
`posAdder := adder()` This call creates a new closure with its own private sum variable (starting at 0). Every call to posAdder(x) modifies its own sum.

`negAdder := adder()` This is a separate call to adder(). It creates a different closure with a different sum variable (also starting at 0). negAdder maintains its own independent state.

```go
package main

import "fmt"

func main() {
    a := func(i int) int { 
        fmt.Println("Inside a(): input =", i, " output =", i*i)
        return i * i 
    } // A function that takes an int and returns its square.

    v := func(f func(i int) int) func() int {
        fmt.Println("Inside v(): Address of f =", &f) // This is the address of the function parameter `f`, which is a local copy of the function `a` passed into `v`.
        c := f(3)
        fmt.Println("Inside v(): Initial c =", c, " Address of c =", &c)

        return func() int {
            fmt.Println("Closure called: Address of c =", &c, " Current Value =", c)
            c++
            fmt.Println("Closure updated: New Value of c =", c)
            return c
        }
    }

    f := v(a) // v() returns a closure // compiler creates closure + environment holding c
    fmt.Println("Inside main: f is", f) // Address of returned closure

    // Call closure multiple times to observe state changes
    fmt.Println("First call result:", f()) // // closure uses c, environment is alive
    fmt.Println("Second call result:", f()) // closure uses c, environment is alive
    fmt.Println("Third call result:", f()) // closure uses c, environment is alive
} // <- f goes out of scope here
// Now closure and c are unreachable // GC will eventually reclaim c
```
```go
func main() {
    {
        f := v(a)        // f exists only inside this block
        fmt.Println(f()) // use it
    } // <- f goes out of scope here
    // At this point, f is unreachable, 
    // Now closure and c are unreachable // GC will eventually reclaim c
}
```

```bash
Inside a(): input = 3  output = 9
Inside v(): Initial c = 9  Address of c = 0xc00011a008
Inside main: f is 0x498740
Closure called: Address of c = 0xc00011a008  Current Value = 9
Closure updated: New Value of c = 10
First call result: 10
Closure called: Address of c = 0xc00011a008  Current Value = 10
Closure updated: New Value of c = 11
Second call result: 11
Closure called: Address of c = 0xc00011a008  Current Value = 11
Closure updated: New Value of c = 12
Third call result: 12

```
- When you call the function v `f := v(a)`, it creates a local variable c by running `f(3)`. Normally, a local variable like `c` would disappear once `v` finishes, but because the function you return from `v` uses `c`, the Go compiler moves `c` from the stack to the heap. This process is called `escape analysis`, and it ensures that `c` stays alive even after `v` returns. 

- The returned closure (`f`) now has a hidden environment object that holds `c`, and every time you call the closure, it looks at the same `c`, prints its value, increments it, and returns the new result. The memory address of `c` stays constant across calls, showing that the closure is holding onto the same variable, while the value changes step by step (for example, 9 → 10 → 11 → 12). 

- As long as you keep a reference to the closure, the garbage collector sees that `c` is still reachable and keeps it alive. Once the closure itself goes out of scope or you set the closure variable to nil, there are no more references to `c`, and the garbage collector can reclaim it. 




## Fan-Out / Fan-In Pattern

Fan-Out :Multiple goroutines read from the same channel and process tasks concurrently. Useful for parallelizing work.

Fan-In :Multiple goroutines send results into a single channel. Useful for aggregating results.

```go
package main

import (
    "fmt"
    "sync"
)

// Worker function
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, j)
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 5)
    results := make(chan int, 5)

    // Fan-Out: start 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // Fan-In: collect results
    for a := 1; a <= 5; a++ {
        fmt.Println("Result:", <-results)
    }
}

```

## Worker Pool Pattern

A fixed number of workers consume tasks from a channel. Prevents spawning too many goroutines. Common in rate-limited or resource-bound systems.

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Printf("Worker %d started job %d\n", id, j)
        time.Sleep(time.Second) // simulate work
        fmt.Printf("Worker %d finished job %d\n", id, j)
        results <- j * j
    }
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int, 10)

    // Start 3 workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Send 5 jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // Collect results
    for a := 1; a <= 5; a++ {
        fmt.Println("Result:", <-results)
    }
}

```

## Pipeline Pattern

Data flows through a series of stages, each stage in its own goroutine. Each stage transforms the data and passes it to the next channel. Useful for streaming transformations.

```go
package main

import "fmt"

// Stage 1: generate numbers
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: square numbers
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: double numbers
func double(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * 2
        }
        close(out)
    }()
    return out
}

func main() {
    nums := gen(1, 2, 3, 4, 5)
    sq := square(nums)
    dbl := double(sq)

    for result := range dbl {
        fmt.Println(result)
    }
}

```

## Wait for Completion Pattern

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// A signal channel to tell the odd routine when even is totally done
	doneWithEven := make(chan bool)
	var wg sync.WaitGroup

	wg.Add(2)

	// Goroutine 1: Even Numbers
	go func() {
		defer wg.Done()
		for i := 0; i <= 30; i += 2 {
			fmt.Printf("Even: %d\n", i)
		}
		// Signal that ALL even numbers are finished
		doneWithEven <- true
	}()

	// Goroutine 2: Odd Numbers
	go func() {
		defer wg.Done()
		// Wait here until a signal is received
		<-doneWithEven
		for i := 1; i <= 30; i += 2 {
			fmt.Printf("Odd:  %d\n", i)
		}
	}()

	wg.Wait()
}
```

## Sequential Polling Pattern

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	evenCh := make(chan int)
	oddCh := make(chan int)

	var wg sync.WaitGroup

	// Producer for Evens
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 1; i <= 10; i++ {
			if i%2 == 0 {
				evenCh <- i
			}
		}
		close(evenCh)
	}()

	// Producer for Odds
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 1; i <= 10; i++ {
			if i%2 != 0 {
				oddCh <- i
			}
		}
		close(oddCh)
	}()

	// Consumer: Reads in pairs (Odd then Even)
	done := make(chan struct{})
	go func() {
		for {
			// Try to read Odd
			odd, okOdd := <-oddCh
			if okOdd {
				fmt.Println("Odd: ", odd)
			}

			// Try to read Even
			even, okEven := <-evenCh
			if okEven {
				fmt.Println("Even:", even)
			}

			// Exit loop only when both channels are closed and drained
			if !okOdd && !okEven {
				break
			}
		}
		close(done)
	}()

	// Wait for producers to finish sending
	wg.Wait()

	// Ensure the consumer has finished printing before the program exits
	<-done
}

```

Preventing Goroutine Leaks: Without ctx.Done(), if your consumer stops listening, the producer goroutine would block forever trying to send data, staying in memory until the program crashes. Graceful Shutdown: It allows goroutines to clean up (close files, finish a database write) before exiting. Propagation: If you pass this ctx to multiple functions, calling cancel() once stops all of them simultaneously.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// Create a context that expires after 50ms
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Microsecond)
	defer cancel() // The function called to trigger the shutdown.

	evenCh := make(chan int)

	go func() {
		i := 0
		for {
			select {
			case <-ctx.Done():
				// Pattern: Stop working if context is cancelled/timed out
				close(evenCh)
				return
			case evenCh <- i:
				i += 2
			}
		}
	}()

	for n := range evenCh {
		fmt.Println("Received:", n)
	}
}

```

In a select statement, a case with a nil channel is disabled. It will never be chosen. Default case will execute. Accidentally reading from a nil channel without a select causes a deadlock:

```go
var ch chan int // nil channel
select {
case <-ch: // <-ch // blocks forever
    fmt.Println("Won't happen")
default:
    fmt.Println("Default case")
}
```

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	// 1. Create a context with cancellation capability
	ctx, cancel := context.WithCancel(context.Background())

	// Ensure cleanup: if we exit main, all goroutines stop
	defer cancel()

	evenCh := make(chan int)
	oddCh := make(chan int)
	var wg sync.WaitGroup

	// 2. Producer: Even numbers with Context check
	wg.Add(1)
	go func() {
		defer wg.Done()
		defer close(evenCh)
		for i := 0; i <= 10; i += 2 {
			select {
			case <-ctx.Done(): // Stop if cancelled
				return
			case evenCh <- i:
			}
		}
	}()

	// 3. Producer: Odd numbers with Context check
	wg.Add(1)
	go func() {
		defer wg.Done()
		defer close(oddCh)
		for i := 1; i <= 10; i += 2 {
			select {
			case <-ctx.Done(): // Stop if cancelled
				return
			case oddCh <- i:
			}
		}
	}()

	// 4. Consumer: Controlled by Context
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			default:
				// Using your sequential polling logic
				vOdd, okOdd := <-oddCh
				if okOdd {
					fmt.Println("Odd: ", vOdd)
				}
				vEven, okEven := <-evenCh
				if okEven {
					fmt.Println("Even:", vEven)
				}
				if !okOdd && !okEven {
					cancel() // Signal completion to others
					return
				}
			}
		}
	}()

	// Simulating a scenario where we might want to stop early
	// (e.g., if a timeout occurred or an error happened elsewhere)
	go func() {
		time.Sleep(1 * time.Second)
		cancel()
	}()

	wg.Wait()
	fmt.Println("System shutdown gracefully.")
}
```

Non-Blocking Shutdown: The select block ensures that if ctx.Done() is closed, the goroutine exits immediately rather than waiting to send a number that no one is listening for.

The defer cancel(): This is a safety net. If main crashes or returns early, it automatically signals all children to stop.

The default Case: In the consumer, the default block allows us to run our polling logic while still keeping an eye on the ctx.Done() signal.
