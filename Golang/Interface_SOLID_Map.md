# Interfaces

Interfaces allow us to define contracts, which are abstract methods, have no body/implementations of the methods. A Struct which wants to implements the Interface need to write the body of every abstract methods the interface holds.

We can compose interfaces together.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
// Embedded interface
type ReadWriter interface {
    Reader
    Writer
}

```

Interafce use for Dependency-Injection means passing the dependency into your function or struct instead of creating it inside. function code doesn’t care which implementation is used — it just uses the interface.

With hardcoded dependencies, testing requires the real system (files, databases, networks).

```go
type FileDB struct{}

func (f FileDB) Save(data string) error {
    fmt.Println("Saving to file:", data)
    return nil
}

func main() {
    fdb := FileDB{}       // hardcoded dependency
    fdb.Save("data")        // directly tied to FileDB
}

```

```go
// Instead of tying your code to a concrete type, you define an interface that describes the behavior you need:
type Database interface {
    Save(data string) error
}

func process(db Database, data string) {
	// Now process depends only on the behavior (Save), not on the specific implementation.
	// "inject" the dependency by passing an implementation of the interface into your function:
    db.Save(data)
}

type FileDB struct{}
func (f FileDB) Save(data string) error {
    fmt.Println("Saving to file:", data)
    return nil
}

type MemoryDB struct{}
func (m MemoryDB) Save(data string) error {
    fmt.Println("Saving to memory:", data)
    return nil
}

func main() {
    fdb := FileDB{} // Object of  File Struct
    mdb := MemoryDB{} // Object of Memory Struct

    process(fdb, "Hello File")   // inject FileDB
    process(mdb, "Hello Memory") // inject MemoryDB
}
```

With interfaces, you can inject a mock or fake implementation. This allows isolated unit tests without external side effects.

If your service depends on a concrete struct (e.g., *repositories.Repo) rather than an interface RepoContarcts, you cannot easily swap it out for a mock in standard Go. This is because Go is statically typed; if a function expects a *Repo, you cannot pass it a *MockRepo.

```go
// Suppose you want to test process. You don’t want to actually write files or hit a real database. You can create a mock implementation:
type MockDB struct {
    Saved []string
}

func (m *MockDB) Save(data string) error {
    m.Saved = append(m.Saved, data)
    return nil
}
func TestProcess(t *testing.T) {
    mock := &MockDB{}
    process(mock, "test-data")

    if len(mock.Saved) != 1 || mock.Saved[0] != "test-data" {
        t.Errorf("Expected 'test-data' to be saved")
    }
}

```

An empty interface can hold any type of values. interface_name.(type) give us the Type the interface will hold at runtime. or we can use reflect.TypeOf(name)

```go
func describe(i interface{}) {
    fmt.Printf("Type: %T, Value: %v\n", i, i)
}

func main() {
    describe(42)
    describe("hello")
    describe(3.14)
}
```

When you use interface{}, you often need to extract the concrete type. This is called Type assertion. It is used to extract the concrete value from an interface type. It tells the Go compiler: I know this interface value is actually of this specific type.

```go
var i interface{} = "golang"

s, ok := i.(string)
if ok {
    fmt.Println("String value:", s)
} else {
    fmt.Println("Not a string")
}

```

An interface value is a pair: (type, value). If the type is set but the value is nil, the interface itself is not nil.

```go
package main

import "fmt"

func main() {
	// 1. A nil pointer of a concrete type (*int)
	var ptr *int = nil

	// 2. Wrap that nil pointer in an empty interface
	var i interface{} = ptr

	// 3. The Comparison
	fmt.Printf("Is ptr nil? %v\n", ptr == nil) // true
	fmt.Printf("Is i nil?   %v\n", i == nil)   // false! <--- The "Gotcha"

	// 4. Let's look at the pair (type, value)
	fmt.Printf("Type: %T, Value: %v\n", i, i) // Type: *int, Value: <nil>
	
	fmt.Println(reflect.TypeOf(i)) // *int
}
// An interface is nil only if both its type and value are nil. If the type is set but the value is nil, the interface is not nil.
```

```go
func main() {
    var i interface{} = 42 // we can change this to any type

    switch reflect.TypeOf(i) {
    case reflect.TypeOf(0):
        fmt.Println("i is an int")
    case reflect.TypeOf(""):
        fmt.Println("i is a string")
    case reflect.TypeOf(true):
        fmt.Println("i is a bool")
    default:
        fmt.Println("Unknown type:", reflect.TypeOf(i))
    }
}
```

# Generics
Generics in Go let you write functions and types that work with many different data types while still being type‑safe. They were introduced in Go 1.18 and are now a core part of the language.
-  Instead of fixing a function to one type (like int), you can declare a parameter T that stands for “any type.”  You can restrict what types are allowed. For example, comparable means the type supports == and !=. The compiler often figures out the type automatically, so you don’t need to specify it explicitly. Generics are compiled down efficiently, so you don’t lose speed compared to writing separate functions.

When you write a generic function like func Print[T any](v T), the compiler doesn't just leave it as a "hole" to be filled at runtime.

Type Checking: The compiler verifies that the types you pass to the generic function satisfy the Type Constraints (e.g., any, comparable, or a custom interface) at compile time.

Code Generation (Stenciling): The compiler generates specialized versions of your function for the different types you use.

If you call Print[int](10) and Print[string]("hi"), the compiler essentially creates two internal versions of that function: one optimized for int and one for string.

Because it is compile-time, you get Full Type Safety. You cannot pass a string into a generic function constrained to constraints.Integer, and the compiler will stop you before the program ever runs.

```go
// T is a type parameter that can be any type
func PrintSlice[T any](s []T) {
    for _, v := range s {
        fmt.Println(v)
    }
}
func main() {
    PrintSlice([]int{1, 2, 3})       // Works with int
    PrintSlice([]string{"a", "b"})   // Works with string
}
```
Types :

- any → allows any type (default if you don’t want restrictions).

- comparable → allows types that support == and !=.

- constraints.Ordered → allows types that can be ordered with <, >, <=, >= (like int, float64, string).

```go
// Comparable constraint allows == and !=
func Index[T comparable](s []T, x T) int {
    for i, v := range s {
        if v == x {
            return i
        }
    }
    return -1
}
func main() {
    fmt.Println(Index([]int{10, 20, 30}, 20))    // 1
    fmt.Println(Index([]string{"a", "b"}, "b")) // 1
}
```
```go
// Define a constraint for numeric types. Here, Number is a custom constraint that allows only int, int64, or float64.
type Number interface {
    int | int64 | float64
}
func Add[T Number](a, b T) T {
    return a + b
}
func main() {
    fmt.Println(Add(10, 20))       // int
    fmt.Println(Add(int64(5), 7))  // int64
    fmt.Println(Add(3.5, 2.1))     // float64
}

```

# SOLID Principles:
SOLID priciples are guidelines for designing Code base that are easy to understand maintain and extend over time.

Single Responsibility:- A Struct/Class should have only a single reason to change. Fields of Author shoud not placed inside Book Struct.
```go
type Book struct{
  ISIN string
  Name String
  AuthorID string
}
type Author struct{
  ID string
  Name String
}
```
Assume One Author decided later, he does not want to Disclose its Real Name to Spread. So we can Serve Frontend by Alias instead of Real Name. Without Changing Book Class/Struct, we can add Alias in Author Struct. By that, Existing Authors present in DB will not be affected as Frontend will Change Name only when it Founds that - Alias field is not empty.
```go
type Book struct{
  ISIN string
  Name String
  AuthorID string
}
type Author struct{
  ID string
  Name String
  Alias String
}

```
Open Close:- Struct and Functions should be open for Extension but closed for modifications. New functionality to be added without changing existing Code.
```go
type Shape interface{
	Area() float64
}
type Rectangle struct{
	W float64
	H float64
}
type Circle struct{
	R float64
}
```
Now we want to Calculate Area of Rectangle and Circle, so Rectangle and Circle both can Implements Shape Interface by Write Body of the Area() Function.
```go
func (r Rectangle) Area()float64{
	return r.W * r.H
}
func (c Circle)Area()float64{
	return 3.14 * c.R * c.R
}
```
Now we can create a Function PrintArea() which take Shape as Arguments and Calculate Area of that Shape. So here Shape can be Rectangle, Circle. In Future we can add Triangle Struct which implements Shape interface by writing Body of Area. Now Traingle can be passed to PrintArea() with out modifing the PrintArea() Function.
```go
func PrintArea(shape Shape) {
	fmt.Printf("Area of the shape: %f\n", shape.Area())
}

// In Future
type Triangle struct{
	B float64
	H float54
}
func (t Triangle)Area()float64{
	return 1/2 * t.B * t.H
}

func main(){
	rect:= Rectangle{W:5,H:3}
	cir:=Circle{R:3}
	PrintArea(rect)
	PrintArea(cir)
	// In Future
	tri:=Triangle{B:4,H:8}
	PrintArea(tri)
}
```
Liskov Substitution:- Super class Object can be replaced by Child Class object without affecting the correctness of the program.
```go
type Bird interface{
	Fly() string
}
type Sparrow struct{
	Name string
}
type Penguin struct{
	Name string
}
```
Sparrow and Pengin both are Bird, But Sparrow can Fly, Penguin Not. ShowFly() function take argument of Bird type and call Fly() function. Now as Penguin and Sparrow both are types of Bird, they should be passed as Bird within ShowFly() function.
```go
func (s Sparrow) Fly() string{
	return "Sparrow is Flying"
}
func (p Penguin) Fly() string{
	return "Penguin Can Not Fly"
}

func ShowFly(b Bird){
	fmt.Println(b.Fly())
}
func main() {
	sparrow := Sparrow{Name: "Sparrow"}
	penguin := Penguin{Name: "Penguin"}
  // SuperClass is Bird,  Sparrow, Penguin are the SubClass
	ShowFly(sparrow)
	ShowFly(penguin)
}
```
Interface Segregation:- A class should not be forced to implements interfaces which are not required for the class. Do not couple multiple interfaces together if not necessary then. 
```go
// The Printer interface defines a contract for printers with a Print method.
type Printer interface {
	Print()
}
// The Scanner interface defines a contract for scanners with a Scan method.
type Scanner interface {
	Scan()
}
// The NewTypeOfDevice interface combines Printer and Scanner interfaces for
// New type of devices which can Print and Scan with it new invented Hardware.
type NewTypeOfDevice interface {
	Printer
	Scanner
}
```

Dependecy Inversion:- Class should depends on the Interfaces not the implementations of methods.

```go
// The MessageSender interface defines a contract for 
//sending messages with a SendMessage method.
type MessageSender interface {
	SendMessage(msg string) error
}
// EmailSender and SMSClient structs implement 
//the MessageSender interface with their respective SendMessage methods.
type EmailSender struct{}

func (es EmailSender) SendMessage(msg string) error {
	fmt.Println("Sending email:", msg)
	return nil
}
type SMSClient struct{}

func (sc SMSClient) SendMessage(msg string) error {
	fmt.Println("Sending SMS:", msg)
	return nil
}
type NotificationService struct {
	Sender MessageSender
}
```
The NotificationService struct depends on MessageSender interface, not on concrete implementations (EmailSender or SMSClient). This adheres to Dependency Inversion, because high-level modules (NotificationService) depend on abstractions (MessageSender) rather than details.
```go
func (ns NotificationService) SendNotification(msg string) error {
	return ns.Sender.SendMessage(msg)
}
func main() {
	emailSender := EmailSender{}

	emailNotification := NotificationService{Sender: emailSender}

	emailNotification.SendNotification("Hello, this is an email notification!")
}
```

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
