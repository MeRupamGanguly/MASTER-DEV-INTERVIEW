
# Map

A map is a built-in data type that stores key-value pairs. Keys must be of a type that is comparable (e.g., strings, integers, booleans). Values can be of any type.

```go
package main

import "fmt"

type User struct {
	Id int
}

func main() {
	m := make(map[struct{}]int)
	//u := User{Id: 2} 
	m[struct{}{}] = 99
	m[struct{}{}] = 98 // Because struct{} has no fields, every instance of struct{}{} is identical to every other instance. Therefore: struct{}{} == struct{}{} is always true. A map[struct{}] can effectively only ever hold one single item.
	fmt.Println(m) // map[{}:98]
	//m[u] = 9  // ./prog.go:15:4: cannot use u (variable of struct type User) as struct{} value in map index

	fmt.Println(m) // map[{}:98]
}

```

`m := make(map[*User]string)` // This map uses the memory address (the pointer) as the key. Even if two users have the exact same Id and Name, they are different keys if they live in different spots in memory

`m := make(map[User]string)` // This map uses the content of the struct to determine the key. If two structs have the exact same data in their fields, they are seen as the same key.

The zero value of a map is nil.
Maps are unordered — iteration order is not guaranteed.
Maps are reference types: assigning or passing them copies the reference, not the data.

Maps are not safe for concurrent use. If multiple goroutines access a map, you need synchronization (e.g., sync.Mutex or sync.Map).

In Go, sync.Map is a specialized, thread-safe map designed for specific use cases where a standard map with a sync.Mutex might be overkill or a performance bottleneck.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// 1. Initialize the map
	var sessions sync.Map

	// 2. Store data (Write)
	// sync.Map uses any (interface{}), so you can store any type.
	// "Store(key, value)",Sets the value for a key.
	sessions.Store("user_1", "active")
	sessions.Store("user_2", "pending")

	// 3. Load data (Read)
	// Load(key),Returns the value and a boolean indicating if it was found.
	val, ok := sessions.Load("user_1")
	if ok {
		// Type assertion is REQUIRED because Load returns any
		fmt.Printf("User 1 status: %v\n", val.(string))
	}

	// 4. LoadOrStore (Atomic "Get or Create")
	// This is great for preventing race conditions when initializing a value
	// "LoadOrStore(key, value)","Returns the existing value if present; otherwise, stores and returns the given value."
	actual, loaded := sessions.LoadOrStore("user_3", "active")
	fmt.Printf("User 3: %v (Already existed: %v)\n", actual, loaded)

	// 5. Iterating (Range)
	// "Range(func(k, v))",Iterates over the map. Iteration stops if the function returns false.
	fmt.Println("Current Sessions:")
	sessions.Range(func(key, value any) bool {
		fmt.Printf(" - %s: %s\n", key, value)
		return true // Return false to stop iteration early
	})

	// 6. Delete
	// Delete(key),Removes a key.
	sessions.Delete("user_2")
}
```

Not all types can be used as map keys.

    Valid: int, string, bool, structs (if all fields are comparable).

    Invalid: slices, maps, functions.

```go
m := make(map[string]int) // keys: string, values: int

m := make(map[string]int, 10) // capacity hint of 10 not a fixed capacity. // The capacity argument in make is only a hint. Maps grow automatically as needed. Unlike slices, maps grow dynamically.

m := map[string]int{} // empty but non-nil map

var m map[string]int // Declares a map variable but it’s nil until initialized. We cannot insert values into a nil map (will panic).

m := map[string]string{
    "name": "Rupam",
    "lang": "Go",
}

```

```go
val, ok := m["age"] // ok is a boolean that tells you if the key exists.
```

Accessing a non-existent key returns the zero value of the value type.

```go
m := map[string]int{"a": 1}
fmt.Println(m["b"]) // prints 0

```

```go
for key, value := range m { // Order is random — Go does not guarantee iteration order.
    fmt.Println(key, value)
}

```

This can cause fatal runtime errors. maps NOT thread-safe

```go
go func() { m["a"] = 1 }()
go func() { fmt.Println(m["a"]) }()

```

Both m1 and m2 point to the same underlying data.

```go
m1 := map[string]int{"a":1}
m2 := m1
m2["a"] = 99
fmt.Println(m1["a"]) // 99
// To truly copy, you must iterate and copy entries manually
```

Delete is safe even if the key doesn’t exist.

```go
m := map[string]int{"a":1}
delete(m, "b") // safe, no error

```

```go
func producer(m map[int]string, wg *sync.WaitGroup, mu *sync.RWMutex) {

	for i := 0; i < 5; i++ {
		mu.Lock()
		m[i] = fmt.Sprint("$", i)
		mu.Unlock()
	}
	wg.Done()
}
func consumer(m map[int]string, wg *sync.WaitGroup, mu *sync.RWMutex) {
	for i := 0; i < 5; i++ {
		mu.RLock()
		fmt.Println(m[i])
		mu.RUnlock()
	}
	wg.Done()
}
func main() {
	m := make(map[int]string)
	m[0] = "1234"
	m[3] = "2345"
	wg := sync.WaitGroup{}
	mu := sync.RWMutex{}
	for i := 0; i < 5; i++ {
		wg.Add(2)
		go producer(m, &wg, &mu)
		go consumer(m, &wg, &mu)
	}
	wg.Wait()
}
```
