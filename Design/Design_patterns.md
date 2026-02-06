# Design Patterns

- Singleton Pattern:  The Singleton pattern ensures a class has only one instance and provides a global point of access to it. The singleton instance is created only when it is first requested, rather than at the time of class loading. It should be ned to be thread-safe, ensuring that multiple threads can safely access the singleton instance concurrently without creating multiple instances. Singletone n pattern can use at Storing and accessing global settings or configurations throughout the application, database connection pool or a logging service where multiple parts of an application need access to a single resource.

```go
package main

import (
    "fmt"
    "sync"
)

// ConfigManager is a singleton struct for managing configuration settings.
type ConfigManager struct {
    config map[string]string //config is a map storing key-value pairs of configuration settings.
    // other fields as needed
}

var (
    instance *ConfigManager
    once     sync.Once
)

// GetConfigManager returns the singleton instance of ConfigManager.
// This function provides global access to the singleton instance of ConfigManager
func GetConfigManager() *ConfigManager {
    once.Do(func() { // sync.Once ensures that, the initialization code, inside once.Do() is executed exactly once, preventing multiple initializations even with concurrent calls.
        instance = &ConfigManager{
            config: make(map[string]string),
        }
        // Initialize configuration settings here
        instance.initConfig()
    })
    return instance
}

// initConfig simulates loading initial configuration settings.
// This method initializes the initial configuration settings. In a real application, this might involve reading settings from a configuration file, a database, or environment variables.
func (cm *ConfigManager) initConfig() {
    // Load configuration settings from file, database, etc.
    cm.config["server_address"] = "localhost"
    cm.config["port"] = "8080"
    // Add more configuration settings as needed
}

// GetConfig retrieves a specific configuration setting.
func (cm *ConfigManager) GetConfig(key string) string {
    return cm.config[key]
}

func main() {
    // Get the singleton instance of ConfigManager
    configManager := GetConfigManager()

    // Access configuration settings
    fmt.Println("Server Address:", configManager.GetConfig("server_address"))
    fmt.Println("Port:", configManager.GetConfig("port"))
}

```

- Builder Pattern:  The Builder pattern in Go (Golang) is used to construct complex objects step by step. 

```go
package main

import "fmt"

// Product represents the complex object we want to build.
type Product struct {
    Part1 string
    Part2 int
    Part3 bool
}

// Builder interface defines the steps to build the Product.
type Builder interface {
    SetPart1(part1 string)
    SetPart2(part2 int)
    SetPart3(part3 bool)
    Build() Product
}

// ConcreteBuilder implements the Builder interface.
type ConcreteBuilder struct {
    part1 string
    part2 int
    part3 bool
}

func (b *ConcreteBuilder) SetPart1(part1 string) {
    b.part1 = part1
}

func (b *ConcreteBuilder) SetPart2(part2 int) {
    b.part2 = part2
}

func (b *ConcreteBuilder) SetPart3(part3 bool) {
    b.part3 = part3
}

func (b *ConcreteBuilder) Build() Product {
    // Normally, here we could implement additional logic or validation
    // before returning the final product.
    return Product{
        Part1: b.part1,
        Part2: b.part2,
        Part3: b.part3,
    }
}

// Director controls the construction process using a Builder.
type Director struct {
    builder Builder
}

func NewDirector(builder Builder) *Director {
    return &Director{builder: builder}
}

func (d *Director) Construct(part1 string, part2 int, part3 bool) Product {
    d.builder.SetPart1(part1)
    d.builder.SetPart2(part2)
    d.builder.SetPart3(part3)
    return d.builder.Build()
}

func main() {
    // Create a concrete builder instance
    builder := &ConcreteBuilder{}

    // Create a director with the concrete builder
    director := NewDirector(builder)

    // Construct the product
    product := director.Construct("Example", 42, true)

    // Print the constructed product
    fmt.Printf("Constructed Product: %+v\n", product)
}
```


- Factory Pattern: Factory pattern is typically used to encapsulate the instantiation of objects, allowing the client code to create objects without knowing the exact type being created.

Here's an example of implementing the Factory pattern with repository selection based on database type:

```go
package main

import (
	"fmt"
	"errors"
)

// Repository interface defines the methods that all concrete repository implementations must implement.
type Repository interface {
	GetByID(id int) (interface{}, error)
	Save(data interface{}) error
	// Add other methods as needed
}

// MySQLRepository represents a concrete implementation of Repository interface for MySQL.
type MySQLRepository struct {
	// MySQL connection details or any necessary configuration
}

func (r *MySQLRepository) GetByID(id int) (interface{}, error) {
	// Implement MySQL specific logic to fetch data by ID
	return nil, errors.New("not implemented")
}

func (r *MySQLRepository) Save(data interface{}) error {
	// Implement MySQL specific logic to save data
	return errors.New("not implemented")
}

// PostgreSQLRepository represents a concrete implementation of Repository interface for PostgreSQL.
type PostgreSQLRepository struct {
	// PostgreSQL connection details or any necessary configuration
}

func (r *PostgreSQLRepository) GetByID(id int) (interface{}, error) {
	// Implement PostgreSQL specific logic to fetch data by ID
	return nil, errors.New("not implemented")
}

func (r *PostgreSQLRepository) Save(data interface{}) error {
	// Implement PostgreSQL specific logic to save data
	return errors.New("not implemented")
}

// RepositoryFactory is the factory that creates different types of repositories based on the database type.
type RepositoryFactory struct{}

func (f *RepositoryFactory) CreateRepository(databaseType string) (Repository, error) {
	switch databaseType {
	case "mysql":
		return &MySQLRepository{}, nil
	case "postgresql":
		return &PostgreSQLRepository{}, nil
	default:
		return nil, fmt.Errorf("unsupported database type: %s", databaseType)
	}
}

func main() {
	factory := RepositoryFactory{}

	// Create a MySQL repository
	mysqlRepo, err := factory.CreateRepository("mysql")
	if err != nil {
		fmt.Println("Error creating MySQL repository:", err)
	} else {
		fmt.Println("Created MySQL repository successfully")
		// Use mysqlRepo as needed
	}

	// Create a PostgreSQL repository
	postgresqlRepo, err := factory.CreateRepository("postgresql")
	if err != nil {
		fmt.Println("Error creating PostgreSQL repository:", err)
	} else {
		fmt.Println("Created PostgreSQL repository successfully")
		// Use postgresqlRepo as needed
	}

	// Try to create a repository with an unsupported database type
	_, err = factory.CreateRepository("mongodb")
	if err != nil {
		fmt.Println("Error creating MongoDB repository:", err)
	}
}
```


- Observer Pattern: The Observer pattern defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.
```go
package main

import (
	"fmt"
	"time"
)

// Observer defines the interface that all observers (subscribers) must implement.
type Observer interface {
	Update(article string)
}

// Subject defines the interface for the subject (publisher) that observers will observe.
type Subject interface {
	Register(observer Observer)
	Deregister(observer Observer)
	Notify(article string)
}

// NewsService represents a concrete implementation of the Subject interface.
type NewsService struct {
	observers []Observer
}

func (n *NewsService) Register(observer Observer) {
	n.observers = append(n.observers, observer)
}

func (n *NewsService) Deregister(observer Observer) {
	for i, obs := range n.observers {
		if obs == observer {
			n.observers = append(n.observers[:i], n.observers[i+1:]...)
			break
		}
	}
}

func (n *NewsService) Notify(article string) {
	for _, observer := range n.observers {
		observer.Update(article)
	}
}

// NewsSubscriber represents a concrete implementation of the Observer interface.
type NewsSubscriber struct {
	Name string
}

func (s *NewsSubscriber) Update(article string) {
	fmt.Printf("[%s] Received article update: %s\n", s.Name, article)
}

func main() {
	// Create a news service
	newsService := &NewsService{}

	// Create news subscribers (observers)
	subscriber1 := &NewsSubscriber{Name: "John"}
	subscriber2 := &NewsSubscriber{Name: "Alice"}
	subscriber3 := &NewsSubscriber{Name: "Bob"}

	// Register subscribers with the news service
	newsService.Register(subscriber1)
	newsService.Register(subscriber2)
	newsService.Register(subscriber3)

	// Simulate publishing new articles
	go func() {
		for i := 1; i <= 5; i++ {
			article := fmt.Sprintf("Article %d", i)
			newsService.Notify(article)
			time.Sleep(time.Second)
		}
	}()

	// Let the program run for a moment to see the output
	time.Sleep(6 * time.Second)

	// Deregister subscriber2
	newsService.Deregister(subscriber2)

	// Simulate publishing another article after deregistration
	article := "Final Article"
	newsService.Notify(article)

	// Let the program run for a moment to see the final output
	time.Sleep(time.Second)
}

```
Advantages of the Observer Pattern:

    Loose coupling: The subject  and observers they don't need to know each other's details beyond the Observer interface.

    Supports multiple observers: The pattern allows for any number of observers to be registered with a subject, and they can all react independently to changes.

    Ease of extension: You can easily add new observers without modifying the subject or other observers.

When to Use the Observer Pattern:

    Event-driven systems: When changes in one object require updates in other related objects (e.g., UI components reflecting changes in underlying data).

    Decoupling behavior: When you want to decouple an abstraction from its implementation so that the two can vary independently.


- Decorator Pattern: The Decorator pattern allows behavior to be added to individual objects, dynamically, without affecting the behavior of other objects from the same class. 

In this example, the Decorator pattern allows us to dynamically add behaviors (enhancements to attack or defense) to an object (the player) without affecting its core functionality. This pattern is useful when you need to extend an object's functionality in a flexible and modular way, which is especially relevant in game development and other software scenarios where objects can have varied and dynamic behaviors.

```go

package main

import "fmt"

// Player interface represents the basic functionalities a player must have.
type Player interface {
	Attack() int
	Defense() int
}

// BasePlayer represents the basic attributes of a player.
type BasePlayer struct {
	attack  int
	defense int
}

func (p *BasePlayer) Attack() int {
	return p.attack
}

func (p *BasePlayer) Defense() int {
	return p.defense
}

// ArmorDecorator enhances a player's defense.
// ArmorDecorator enhances a player's defense by adding an additional armor value.
type ArmorDecorator struct {
	player Player
	armor  int
}

func (a *ArmorDecorator) Attack() int {
	return a.player.Attack()
}

func (a *ArmorDecorator) Defense() int {
	return a.player.Defense() + a.armor
}

// WeaponDecorator enhances a player's attack.
// WeaponDecorator enhances a player's attack by adding an additional attack value.
type WeaponDecorator struct {
	player Player
	attack int
}

func (w *WeaponDecorator) Attack() int {
	return w.player.Attack() + w.attack
}

func (w *WeaponDecorator) Defense() int {
	return w.player.Defense()
}

func main() {
	// Create a base player
	player := &BasePlayer{
		attack:  10,
		defense: 5,
	}

	fmt.Println("Base Player:")
	fmt.Printf("Attack: %d, Defense: %d\n", player.Attack(), player.Defense())

	// Add armor to the player
	playerWithArmor := &ArmorDecorator{
		player: player,
		armor:  5,
	}

	fmt.Println("Player with Armor:")
	fmt.Printf("Attack: %d, Defense: %d\n", playerWithArmor.Attack(), playerWithArmor.Defense())

	// Add a weapon to the player with armor
	playerWithArmorAndWeapon := &WeaponDecorator{
		player: playerWithArmor,
		attack: 8,
	}

	fmt.Println("Player with Armor and Weapon:")
	fmt.Printf("Attack: %d, Defense: %d\n", playerWithArmorAndWeapon.Attack(), playerWithArmorAndWeapon.Defense())
}
```
```bash
# OUTPUT
Base Player:
Attack: 10, Defense: 5
Player with Armor:
Attack: 10, Defense: 10
Player with Armor and Weapon:
Attack: 18, Defense: 10
```
- Strategy Pattern: The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it. The strategy pattern is a behavioral design pattern that enables selecting an algorithm at runtime from a family of algorithms. It allows a client to choose from a variety of algorithms or strategies without altering their structure. This pattern is useful when you have multiple algorithms that can be interchangeable depending on the context.

```go
// Defines a contract for sorting algorithms. Any concrete sorting algorithm (BubbleSort, QuickSort) must implement this interface.
type SortStrategy interface {
    Sort([]int) []int
}
```
```go
ype BubbleSort struct{}

func (bs *BubbleSort) Sort(arr []int) []int {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
    return arr
}
```

```go
type QuickSort struct{}

func (qs *QuickSort) Sort(arr []int) []int {
    if len(arr) < 2 {
        return arr
    }
    pivot := arr[0]
    var less, greater []int
    for _, v := range arr[1:] {
        if v <= pivot {
            less = append(less, v)
        } else {
            greater = append(greater, v)
        }
    }
    sorted := append(append(qs.Sort(less), pivot), qs.Sort(greater)...)
    return sorted
}

```
```go
// Holds a reference to a strategy object (SortStrategy) and provides methods to set and use different sorting algorithms. It delegates the sorting task to the current strategy.
type SortContext struct {
    strategy SortStrategy
}

func NewSortContext(strategy SortStrategy) *SortContext {
    return &SortContext{strategy: strategy}
}

func (sc *SortContext) SetStrategy(strategy SortStrategy) {
    sc.strategy = strategy
}

func (sc *SortContext) SortArray(arr []int) []int {
    return sc.strategy.Sort(arr)
}

```
```go
func main() {
    arr := []int{64, 25, 12, 22, 11}

    // Use Bubble Sort
    bubbleSort := &BubbleSort{}
    context := NewSortContext(bubbleSort)
    sorted := context.SortArray(arr)
    fmt.Println("Sorted using Bubble Sort:", sorted)

    // Use Quick Sort
    quickSort := &QuickSort{}
    context.SetStrategy(quickSort)
    sorted = context.SortArray(arr)
    fmt.Println("Sorted using Quick Sort:", sorted)
}
```
The strategy pattern allows the client (main function) to select different algorithms dynamically at runtime.
Algorithms are encapsulated in their own classes (BubbleSort, QuickSort), adhering to the Single Responsibility Principle.
Adding new sorting algorithms (MergeSort, InsertionSort, etc.) involves creating new classes that implement SortStrategy.
