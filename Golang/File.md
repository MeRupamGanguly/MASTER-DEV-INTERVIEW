
# File

In computing, a file is a shared resource. When multiple goroutines (threads) access a single file, we encounter two main challenges:

Race Conditions: If two writers try to write to the same spot at the same time, the data can become interleaved (e.g., "WriWriter 1ter 2").

File Offsets: Every open file descriptor has a "cursor" (offset). If Goroutine A reads 10 bytes, the cursor moves. If Goroutine B then tries to read, it starts from byte 11, not byte 0.

The Snapshot vs. Stream Mental Model
Snapshotting: Think of this like taking a photograph. You want to see the whole file exactly as it exists at 12:00 PM. This requires "freezing" the writers briefly so the photo isn't blurry.

Tailing: Think of this like a security camera feed. You don't care about the whole history at once; you just want to see every new "event" as it happens, one frame (or line) at a time.

The simplest way to create a file is os.Create. If the file already exists, it will be truncated (emptied).

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// Create a new file
	file, err := os.Create("example.txt")
	if err != nil {
		fmt.Println("Error creating file:", err)
		return
	}
	// Always defer Close to ensure resources are freed
	defer file.Close()

	// Write a string
	n, err := file.WriteString("Hello Go File System!\n")
	if err != nil {
		fmt.Println("Error writing string:", err)
		return
	}
	fmt.Printf("Wrote %d bytes\n", n)

	data, err := os.ReadFile("example.txt")
	if err != nil {
		fmt.Println("Error reading file:", err)
		return
	}
	fmt.Println("File content:", string(data))
}
```
```go
// O_APPEND: Add to end
// O_WRONLY: Open for writing only
// O_CREATE: Create if it doesn't exist
f, err := os.OpenFile("example.txt", os.O_APPEND|os.O_WRONLY|os.O_CREATE, 0644)
if err != nil {
    panic(err)
}
defer f.Close()

f.WriteString("New log entry\n")
```
For large files (GBs), you shouldn't load the whole file into memory. Use bufio.Scanner to read it line by line.

```go
file, _ := os.Open("large_file.log")
defer file.Close()

scanner := bufio.NewScanner(file)
for scanner.Scan() {
    fmt.Println("Line:", scanner.Text())
}

if err := scanner.Err(); err != nil {
    fmt.Println("Error scanning:", err)
}
```
## Concurrent logging and log-reading
```go
package main

import (
	"bufio"
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"sync"
	"time"
)

type MyLogger struct {
	logg *log.Logger
	MU   sync.Mutex
}

func (l *MyLogger) LOG(msg string) {
	l.MU.Lock()
	l.logg.Print(msg)
	l.MU.Unlock()
}

func NewMyLogger(f string) *MyLogger {
	file, err := os.OpenFile(f, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	if err != nil {
		panic(err)
	}
	return &MyLogger{
		logg: log.New(file, "", log.LstdFlags),
	}
}

func ReadLog(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	f, err := os.OpenFile("m_log.log", os.O_CREATE, 0644)
	if err != nil {
		panic(err)
	}
	reading := bufio.NewReader(f)
	for {
		line, err := reading.ReadString('\n')
		if err != nil {
			if errors.Is(err, io.EOF) {
				break
			}
			fmt.Println("err")
			time.Sleep(10 * time.Millisecond)
		} else {
			fmt.Println("Reader Name ", id, "Reading", line)
			time.Sleep(120 * time.Millisecond)
		}
	}
}

func main() {
	l := NewMyLogger("m_log.log")
	wg := sync.WaitGroup{}
	for i := range 3 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range 20 {
				msg := fmt.Sprint("Hi this is log from GOROUTINE NUMBER: ", i, " Line Number: ", j)
				l.LOG(msg)
				time.Sleep(100 * time.Millisecond)
			}
		}()

		time.Sleep(time.Second)
		wg.Add(3)
		go ReadLog(1, &wg)
		go ReadLog(2, &wg)
		go ReadLog(3, &wg)
	}
	wg.Wait()
}
```

Instead of multiple goroutines writing to the same file and locking with a mutex, you can send log messages into a channel.

```go
package main

import (
	"bufio"
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"sync"
	"time"
)

type MyLogger struct {
	logg *log.Logger
	ch   chan string
}

func (l *MyLogger) LOGTOFILE(wg *sync.WaitGroup) {
	defer wg.Done()
	for d := range l.ch {
		l.logg.Print(d)
	}
}

func NewMyLogger(f string) *MyLogger {
	file, err := os.OpenFile(f, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	if err != nil {
		panic(err)
	}
	return &MyLogger{
		logg: log.New(file, "", log.LstdFlags),
		ch:   make(chan string, 100),
	}
}

func ReadLog(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	f, err := os.OpenFile("m_log.log", os.O_CREATE, 0644)
	if err != nil {
		panic(err)
	}
	reading := bufio.NewReader(f)
	for {
		line, err := reading.ReadString('\n')
		if err != nil {
			if errors.Is(err, io.EOF) {
				break
			}
			fmt.Println("err")
			time.Sleep(10 * time.Millisecond)
		} else {
			fmt.Println("Reader Name ", id, "Reading", line)
			time.Sleep(120 * time.Millisecond)
		}
	}
}

func main() {
	l := NewMyLogger("m_log.log")
	wg := sync.WaitGroup{}
	wgW := sync.WaitGroup{}
	wgW.Add(1)
	go l.LOGTOFILE(&wgW)
	for i := range 3 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range 20 {
				msg := fmt.Sprint("Hi this is log from GOROUTINE NUMBER: ", i, " Line Number: ", j)
				l.ch <- msg
				time.Sleep(100 * time.Millisecond)
			}
		}()

		time.Sleep(time.Second)
		wg.Add(3)
		go ReadLog(1, &wg)
		go ReadLog(2, &wg)
		go ReadLog(3, &wg)
	}
	wg.Wait()
	go func() {
		wgW.Wait()
		close(l.ch)
	}()
}
```

```go
package main

import (
	"bufio"
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"sync"
	"time"
)

type MyLogger struct {
	logg *log.Logger
	ch   chan string
	wg   *sync.WaitGroup
}

func (l *MyLogger) LOGTOFILE() {
	defer l.wg.Done()
	for d := range l.ch {
		l.logg.Print(d)
	}
}
func (l *MyLogger) Close() {
	close(l.ch)
	l.wg.Wait()
}

func NewMyLogger(f string) *MyLogger {
	file, err := os.OpenFile(f, os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	if err != nil {
		panic(err)
	}

	mylo := MyLogger{
		logg: log.New(file, "", log.LstdFlags),
		ch:   make(chan string, 100),
		wg:   &sync.WaitGroup{},
	}
	mylo.wg.Add(1)
	go mylo.LOGTOFILE()
	return &mylo
}

func ReadLog(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	f, err := os.OpenFile("m_log.log", os.O_CREATE, 0644)
	if err != nil {
		panic(err)
	}
	reading := bufio.NewReader(f)
	for {
		line, err := reading.ReadString('\n')
		if err != nil {
			if errors.Is(err, io.EOF) {
				break
			}
			fmt.Println("err")
			time.Sleep(10 * time.Millisecond)
		} else {
			fmt.Println("Reader Name ", id, "Reading", line)
			time.Sleep(120 * time.Millisecond)
		}
	}
}

func main() {
	l := NewMyLogger("m_log.log")
	wg := sync.WaitGroup{}
	for i := range 3 {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range 20 {
				msg := fmt.Sprint("Hi this is log from GOROUTINE NUMBER: ", i, " Line Number: ", j)
				l.ch <- msg
				time.Sleep(100 * time.Millisecond)
			}
		}()

		time.Sleep(time.Second)
		wg.Add(3)
		go ReadLog(1, &wg)
		go ReadLog(2, &wg)
		go ReadLog(3, &wg)
	}
	wg.Wait()
	l.Close()
}
```
Multiple workers "write" to the channel simultaneously without waiting, but the channel forces those messages into a single-file line so the actual file is written one-by-one.

The Many (Workers): Run in parallel, firing messages into the channel buffer.

The One (Channel): Automatically serializes (orders) those messages.

The Single (Writer): Picks up the ordered messages one-by-one and writes them to the disk.

By using a channel as a buffer, your "worker" goroutines don't have to wait for the slow disk to finish writing. They just drop the message in the channel and move on. This significantly improves performance in high-concurrency systems. Instead of multiple goroutines fighting over a file lock, they all send data to a single channel, and one dedicated goroutine handles all the disk I/O. They only block if the logCh buffer (size 100) gets completely full.

Go channels are internally synchronized. This means all 5 workers can try to "push" messages into the channel at the exact same time, and the channel will handle the queueing safely without any extra code from you.

Because the channel is buffered (size 100), workers can work ahead of the disk writer. However, if the disk becomes very slow and 100 messages pile up, the workers will naturally slow down (block) until the writer clears space.

bufio.NewWriter creates a small slice of RAM. Writing to RAM is nanoseconds; writing to a Hard Drive is milliseconds. The worker messages are packed into that RAM buffer. When the buffer is full (or when bufferedWriter.Flush() is called at the end), the OS performs one big write operation instead of 50 small ones. This reduces "System Call" overhead.

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"sync"
)
// We use bufio.NewWriter. Instead of hitting the disk for every single string, it collects them in memory and writes them in one large, efficient "chunk."
func main() {
	// 1. Create a channel for logs
	logCh := make(chan string, 100) // Buffered channel for better performance
	var wg sync.WaitGroup

	// 2. Open the file for writing
	file, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		panic(err)
	}
	defer file.Close()

	// 3. Start the Dedicated Writer Goroutine
	// This is the ONLY goroutine that touches the file. No Mutex needed!
	writerDone := make(chan struct{})
	go func() {
		// Use bufio for faster, batched writes to disk
		bufferedWriter := bufio.NewWriter(file)
		for msg := range logCh {
			bufferedWriter.WriteString(msg + "\n")
		}
		bufferedWriter.Flush() // Ensure everything is written before closing
		close(writerDone)
	}()

	// 4. Start multiple worker goroutines (Producers)
	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := 1; j <= 10; j++ {
				logCh <- fmt.Sprintf("Worker %d: logic step %d", id, j)
			}
		}(i)
	}

	// 5. Wait for workers to finish, then close the channel
	wg.Wait()
	close(logCh)

	// 6. Wait for the writer to finish draining the channel
	<-writerDone // The main function pauses here. It refuses to let the program die until the writer has finished writing every single remaining message and called .Flush().
	fmt.Println("All logs written to app.log successfully.")
}
```
Shared Access, Separate Offsets: Because os.Open is used inside the goroutine, "Alpha" doesn't care where "Beta" is reading. Alpha can be at line 10 while Beta is at line 2. The OS manages this perfectly.

No Locking Required: Since these goroutines are only Reading, they are not changing the file. In Go (and most OSs), reading is a "non-blocking" operation for other readers.

Parallelism: If you have a multi-core CPU, these three readers are literally scanning the file at the exact same time.

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
	"sync"
	"time"
)

func reader(name string, wg *sync.WaitGroup) {
	defer wg.Done()

	// Each reader opens its own handle to the file.
	// This gives this reader its own private "bookmark" (offset).
	file, err := os.Open("app.log")
	if err != nil {
		fmt.Printf("Reader %s error: %v\n", name, err)
		return
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	lineCount := 0

	for scanner.Scan() {
		lineCount++
		// We print just the count to keep the console clean
		// but in reality, all readers are seeing all lines.
		fmt.Printf("[%s] read line %d: %s\n", name, lineCount, scanner.Text())
		
		// Small sleep to simulate processing time and show concurrency
		time.Sleep(10 * time.Millisecond)
	}

	if err := scanner.Err(); err != nil {
		fmt.Printf("Reader %s scan error: %v\n", name, err)
	}
	fmt.Printf(">>> Reader %s finished reading %d lines.\n", name, lineCount)
}

func main() {
	var wg sync.WaitGroup

	// Let's start 3 readers at the same time
	readerNames := []string{"Alpha", "Beta", "Gamma"}

	for _, name := range readerNames {
		wg.Add(1)
		go reader(name, &wg)
	}

	wg.Wait()
	fmt.Println("Main: All concurrent readers have finished.")
}
```
