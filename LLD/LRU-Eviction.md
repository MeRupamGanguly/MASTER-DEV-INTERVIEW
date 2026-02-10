# LRU Eviction Policy

• In-Memory Storage: The cache must store key-value pairs in memory using appropriate data structures.
• LRU Eviction Policy: When the cache reaches its maximum capacity, it must remove the least recently used (oldest) entry to make space for new data.
• APIs: The system must expose two primary methods: Put(key, value) and Get(key).
• Extensibility: The design should be "loosely coupled" so that storage (e.g., switching to a database) or eviction policies (e.g., switching to FIFO) can be changed without rewriting the core logic

Below is the implementation in Go.

### 1. Interfaces
Following the "dependency inversion" principle mentioned in the sources, we define interfaces for storage and eviction so that the cache remains extensible (e.g., adding a database-backed storage later).


 It uses the container/list package to implement the doubly linked list required for tracking the access order of keys [conversation history].

```go
package main

import (
	"container/list" // It’s part of Go’s standard library.
	"fmt"
)

// CacheStorage defines how data is stored and retrieved
type CacheStorage interface {
	Get(key string) (string, bool)
	Put(key, value string)
	Remove(key string)
	IsFull() bool
}

// CacheEvictionStrategy defines the policy for removing keys
type CacheEvictionStrategy interface {
	EvictKey() string
	KeyAccessed(key string)
}
```

### 2. The Main Cache Struct
The `Cache` struct depends on the interfaces, not concrete implementations, making it "loosely coupled".

```go
type Cache struct {
	storage          CacheStorage
	evictionStrategy CacheEvictionStrategy
}

func NewCache(storage CacheStorage, strategy CacheEvictionStrategy) *Cache {
	return &Cache{
		storage:          storage,
		evictionStrategy: strategy,
	}
}

func (c *Cache) Get(key string) (string, bool) {
	val, ok := c.storage.Get(key)
	if ok {
		// Even for Get, we must notify the strategy that the key was used
		c.evictionStrategy.KeyAccessed(key)
	}
	return val, ok
}

func (c *Cache) Put(key, value string) {
	// If full and it's a new key, evict the oldest entry
	_, exists := c.storage.Get(key)
	if c.storage.IsFull() && !exists {
		keyToEvict := c.evictionStrategy.EvictKey()
		c.storage.Remove(keyToEvict)
	}

	c.storage.Put(key, value)
	c.evictionStrategy.KeyAccessed(key) // Update access order
}
```

### 3. In-Memory Storage Implementation
This implementation uses a standard Go `map` to store key-value pairs in memory.

```go
type InMemoryStorage struct {
	data     map[string]string
	capacity int
}

func NewInMemoryStorage(capacity int) *InMemoryStorage {
	return &InMemoryStorage{
		data:     make(map[string]string),
		capacity: capacity,
	}
}

func (s *InMemoryStorage) Get(key string) (string, bool) {
	val, ok := s.data[key]
	return val, ok
}

func (s *InMemoryStorage) Put(key, value string) {
	s.data[key] = value
}

func (s *InMemoryStorage) Remove(key string) {
	delete(s.data, key)
}

func (s *InMemoryStorage) IsFull() bool {
	return len(s.data) >= s.capacity
}
```

### 4. LRU Eviction Strategy Implementation
This uses a `container/list` to track the access order (Double Linked List) and a `map` to track existing keys for efficiency.

```go
type LRUEvictionStrategy struct {
	accessOrder  *list.List //  accessOrder: The linked list that tracks the order of key usage.
	existingKeys map[string]*list.Element // A map from key → pointer to its node (list.Element) in the linked list.
// This allows O(1) lookup of where a key is in the list.
}
/*
 * Each element (list.Element) has pointers to:
 
 The previous element
 
 The next element
 
 Its value (the data you store)
 */
func NewLRUEvictionStrategy() *LRUEvictionStrategy {
	return &LRUEvictionStrategy{
		accessOrder:  list.New(), //  creates an empty doubly linked list.
		existingKeys: make(map[string]*list.Element), // creates a hash map to quickly find elements.
	}
}

func (l *LRUEvictionStrategy) KeyAccessed(key string) {
	// If key exists, move it to the back (most recent)
	if element, ok := l.existingKeys[key]; ok {
		l.accessOrder.MoveToBack(element) //  moves it to the end of the list, marking it as most recently used.
	} else {
		// New key: add to the back
		element := l.accessOrder.PushBack(key) // adds it to the end. // Store the pointer in the map for fast lookup later.
		l.existingKeys[key] = element
	}
}

func (l *LRUEvictionStrategy) EvictKey() string {
	// Remove from the front (least recently used)
	element := l.accessOrder.Front() //  gets the least recently used element (the head of the list).
	if element == nil {
		return ""
	}
	key := element.Value.(string)
	l.accessOrder.Remove(element)
	delete(l.existingKeys, key)
	return key
}
```
When you call MoveToBack, PushBack, or Remove, the library updates these pointers (next, prev) so the linked list stays consistent.

Operations like moving or removing are O(1) because they only adjust pointers, not shift memory like slices.

### 5. Demo Execution
This mirrors the test case provided in the sources where a capacity of 3 is used and the oldest key is evicted when a 4th key is added.

```go
func main() {
	capacity := 3
	storage := NewInMemoryStorage(capacity)
	strategy := NewLRUEvictionStrategy()
	cache := NewCache(storage, strategy)

	cache.Put("1", "Apple")
	cache.Put("2", "Ball")
	cache.Put("3", "Cat")

	// This should trigger eviction of "1" (Apple)
	cache.Put("4", "Dog")

	val, ok := cache.Get("1")
	if !ok {
		fmt.Println("Key 1: Not found (Evicted successfully)")
	} else {
		fmt.Println("Key 1:", val)
	}

	val4, _ := cache.Get("4")
	fmt.Println("Key 4:", val4)
}
```

### Key Considerations from the Sources:
*   **Loose Coupling:** The `Cache` struct does not know if it's using in-memory or a database; it only knows the `CacheStorage` interface.
*   **LRU Logic:** The "Tail" of our list represents the most recently used item, while the "Head" represents the oldest, which is evicted first.
*   **Thread Safety:** As noted in the sources, this basic implementation is **not thread-safe**. To make it thread-safe in Go (outside the scope of the provided sources but mentioned as a future step), you would typically wrap these operations in a `sync.Mutex`.
*   **Generics:** While Go supports generics, this implementation uses `string` to match the initial draft described in the source.

---

To make the LRU Cache **thread-safe**, we must ensure that multiple goroutines can call `Get` and `Put` simultaneously without causing race conditions or data corruption. The sources explicitly identify "thread safety" as a crucial requirement for using the cache in a **multi-threaded environment**.

While the sources mention that concurrency management is a necessary advancement, the transcript concludes before providing the specific implementation details for it. However, in Go, the standard way to achieve the "concurrency management" mentioned in the sources is by using a **Mutex** (Mutual Exclusion lock).

### **How to Implement Thread Safety**
To protect the internal state (the map and the linked list), we need to lock the entire sequence of operations in `Get` and `Put`. This is because even a `Get` operation modifies the "access order" in the eviction strategy to maintain the LRU property.

Here is the thread-safe version of the `Cache` struct in Go:

```go
package main

import (
	"sync" // Required for Mutex
)

type Cache struct {
	storage          CacheStorage
	evictionStrategy CacheEvictionStrategy
	mu               sync.Mutex // The lock that ensures thread safety
}

func NewCache(storage CacheStorage, strategy CacheEvictionStrategy) *Cache {
	return &Cache{
		storage:          storage,
		evictionStrategy: strategy,
	}
}

func (c *Cache) Get(key string) (string, bool) {
	// Acquire the lock before any operations
	c.mu.Lock()
	// Ensure the lock is released when the function returns
	defer c.mu.Unlock()

	val, ok := c.storage.Get(key)
	if ok {
		// Sources note that Get must update the access order
		c.evictionStrategy.KeyAccessed(key)
	}
	return val, ok
}

func (c *Cache) Put(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	// Logic from sources: check if full and if key is new
	_, exists := c.storage.Get(key)
	if c.storage.IsFull() && !exists {
		// Identify and remove the oldest key
		keyToEvict := c.evictionStrategy.EvictKey()
		c.storage.Remove(keyToEvict)
	}

	c.storage.Put(key, value)
	c.evictionStrategy.KeyAccessed(key)
}
```

### **Why this is necessary (Based on Sources)**
1.  **Race Conditions on Access Order:** The sources explain that LRU requires moving a key to the "tail" (most recent) every time it is accessed. If two threads call `Get` at the same time without a lock, they might both try to modify the pointers in the doubly linked list simultaneously, leading to a crash or corrupted list.
2.  **Consistency Between Map and List:** The "Storage" (map) and "Eviction Strategy" (list) must remain in sync. Without a lock, one thread could evict a key from the list while another thread is still trying to read it from the map.
3.  **Check-and-Act Atomicity:** In the `Put` method, we first check `IsFull()` and then perform an `EvictKey()`. This "check-then-act" sequence must be **atomic**. If two threads see the cache is "almost full" at the same time, they might both try to add an item without evicting, exceeding the cache capacity.

***Note on Implementation:** The use of `sync.Mutex` is a standard Go practice for the "concurrency management" described in the sources. Using `defer c.mu.Unlock()` is the idiomatic way to ensure the lock is always released, even if the function encounters an error or returns early.*
