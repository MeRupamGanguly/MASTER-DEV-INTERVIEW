# Array
An array is a fixed-size, contiguous collection of elements of the same type.
var arr [N]T where N is length and T is type.
Length is part of type: [3]int and [4]int are different types.
Arrays are copied on assignment and passed by value to functions.
var arr [3]int → [0 0 0]

arr := [3]int{1,2,3}

arr := [...]int{1,2,3} → compiler infers length.
```go
a := [3]int{1}
fmt.Println(a) // [1 0 0]
```
```go
func change(arr [3]int) { // passed by value 
	arr[1] = 0
}
func main() {
	a := [3]int{1, 2, 3}
	change(a)
	fmt.Println(a) // [1 2 3]
}
```
```go
func change(arr *[3]int) {
	arr[1] = 0
}
func main() {
	a := [3]int{1, 2, 3}
	change(&a)
	fmt.Println(a) // [1 0 3]
}
```
Arrays only comparable if element type is comparable.
```go
var a [3]int
var b [4]int
a = b //  compile error: cannot use b (variable of type [4]int) as [3]int value in assignment.
```
```go
var a [2][]int
var b [2][]int
fmt.Println(a==b) //  invalid operation: a == b ([2][]int cannot be compared)

```
```go
func main() {
	var a [2][3]int
	var b [2][3]int
	fmt.Println(a == b) // True
}

```
```go
func main() {
	var a [2]int
	var b [2]int
	fmt.Println(a == b) // True
}
```

```go
func main() {
	a := [3]int{1, 2, 3}
	b := [4]int{1, 2, 4, 5}
	fmt.Println(a == b) // invalid operation: a == b (mismatched types [3]int and [4]int)
}
```
```go
func main() {
	a := [3]int{1, 2, 3}
	b := [3]int{1, 4, 5}
	fmt.Println(a == b) // false // Not compile error
}
```
```go
a := [3]int{1,2,3}
b := a
b[0] = 99
fmt.Println(a) // [1 2 3] unchanged
```

```go
arr := [3]int{1,2,3}
s := arr[:] // slice referencing array
s[0] = 99
fmt.Println(arr) // [99 2 3] modified
```
```go
func f(x [4]int){}
a := [3]int{1,2,3}
f(a) // compile error: type mismatch
// cannot use a (variable of type [3]int) as [4]int value in argument to f
```
```go
a := [2]int{1,2,3} // compile error: too many values

```
```go
arr := [3]int{1, 2, 3}
for _, v := range arr {
	fmt.Println(&v, v) // same address reused
}
/*
0xc000108008 1
0xc000108030 2
0xc000108038 3
*/
```

```go
a := [3][3]int{{1, 2}, {4, 5, 6}} // [3][3] array of array
fmt.Println(a) // [[1 2 0] [4 5 6] [0 0 0]]

a := [3][2]int{{1, 2}, {4, 5}, {8, 9}}
fmt.Println(a) // [[1 2] [4 5] [8 9]]
```
```go
func main() {
	a := [3][]int{{1, 2}, {4, 5}, {8, 9}} // [empty] make it array of slice
	b := a
	b[1][1] = 0
	fmt.Println(a) // [[1 2] [4 0] [8 9]]
}
```
```go
a := [3][3]int{{1, 2}, {4, 5}, {8, 9}}
b := &a
b[1][1] = 99
fmt.Println(a) // [[1 2 0] [4 99 0] [8 9 0]]
```
```go
func main() {
	a := [3]int{1, 2, 3}
	for _, v := range a {
		v = v * 10
	}
	fmt.Println(a) // [1 2 3] both array and slice same output
}
```
```go
func main() {
    type myarr [3]int
    var a myarr = [3]int{5, 7, 9}

    fmt.Printf("Type of a: %T\n", a)
}
func main() {
    type myarr [3]int
    var a myarr = [3]int{5, 7, 9}

    fmt.Println("Type of a:", reflect.TypeOf(a))
}
// Type of a: main.myarr
// Defining type myarr [3]int creates a named type, so variables of that type are not interchangeable with plain [3]int unless explicitly converted.
// myarr is a distinct type from [3]int. Even though it has the same underlying representation, Go treats it as a new type.

func main() {
	var myarr [3]int
	var a myarr = [3]int{5, 7, 9} // Panic: myarr is not a type
	fmt.Printf("Type of a: %T\n", a)
}
```
```go
func main() {
	a := [3]int{1, 2, 3}
	b := [3]int{4, 5, 6}
	a = b
	fmt.Println(a)
}
```

```go
func main() {
	a := [3]int{1, 2, 3}
	b := &[3]int{4, 5, 6}
	a = *b
	fmt.Println(a)
}
```

# Slice

A slice is like a flexible view of an array.

A slice in Go is a descriptor that contains three things:
- Pointer → points to the first element of the underlying array.
- Length → number of elements currently in the slice.
- Capacity → maximum number of elements the slice can hold before needing a new array.

It doesn’t hold data itself, but points to an underlying array. Unlike arrays, slices can grow and shrink using append or slicing operations. Slices are reference types → passing a slice to a function does not copy all elements, it just passes a reference. This makes them more efficient. 

```go
func main() {
	var s []int
	s2 := []int{}
	s3 := make([]int, 3)
	fmt.Println(s, s2, s3) // [] [] [0 0 0]
	fmt.Println(s == nil, s2 == nil) // true false
}
```

Multiple slices can point to the same array. Changing one slice may affect another. When append exceeds capacity, Go allocates a new array (usually doubling size). The slice then points to this new array.

A slice declared without initialization (var s []int) is nil, but still safe to use with append.


```go
func main() {
	a := []int{1, 2, 3}
	b := a[:2]
	fmt.Println(a, b) // [1 2 3] [1 2]
	c := a[1:]
	fmt.Println(a, b, c) // [1 2 3] [1 2] [2 3]
	b[1] = 99
	fmt.Println(a, b, c) // [1 99 3] [1 99] [99 3]
}
func main() {
	a := []int{1, 2, 3}
	b := a[:2]
	fmt.Println(a, b) // [1 2 3] [1 2]
	c := a[1:]
	fmt.Println(a, b, c) // [1 2 3] [1 2] [2 3]
	b = append(b, 4, 5, 6) 
	b[1] = 99
	fmt.Println(a, b, c) // [1 2 3] [1 99 4 5 6] [2 3]
}
```

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	a := []int{1, 2, 3}
	b := a[:2]
	c := a[1:]

	fmt.Printf("a: %v, addr(slice header): %p, addr(backing array): %p-%p-%p\n", a, &a, &a[0], &a[1], &a[2])
	fmt.Printf("b: %v, addr(slice header): %p, addr(backing array): %p-%p\n", b, &b, &b[0], &b[1])
	fmt.Printf("c: %v, addr(slice header): %p, addr(backing array): %p-%p\n", c, &c, &c[0], &c[1])
	fmt.Printf("c backing array base: %p\n", unsafe.Pointer(&c[:cap(c)][0]))

	b[1] = 99
	fmt.Println("After modification:")
	fmt.Printf("a: %v, addr(backing array): %p\n", a, &a[0])
	fmt.Printf("b: %v, addr(backing array): %p\n", b, &b[0])
	fmt.Printf("c: %v, addr(backing array): %p\n", c, &c[0])

	// Now append to b
	b = append(b, 4, 5, 6)
	fmt.Println("After append:")
	fmt.Printf("a: %v, addr(backing array): %p\n", a, &a[0])
	fmt.Printf("b: %v, addr(backing array): %p\n", b, &b[0])
	fmt.Printf("c: %v, addr(backing array): %p\n", c, &c[0])
}
```

```bash
a: [1 2 3], addr(slice header): 0xc000010048, addr(backing array): 0xc00001a018-0xc00001a020-0xc00001a028
b: [1 2], addr(slice header): 0xc000010060, addr(backing array): 0xc00001a018-0xc00001a020
c: [2 3], addr(slice header): 0xc000010078, addr(backing array): 0xc00001a020-0xc00001a028
c backing array base: 0xc00001a020
After modification:
a: [1 99 3], addr(backing array): 0xc00001a018
b: [1 99], addr(backing array): 0xc00001a018
c: [99 3], addr(backing array): 0xc00001a020
After append:
a: [1 99 3], addr(backing array): 0xc00001a018
b: [1 99 4 5 6], addr(backing array): 0xc000106000
c: [99 3], addr(backing array): 0xc00001a020
```
a = []int{1,2,3} → slice header points to &a[0].

b = a[:2] → also points to &a[0].

c = a[1:] → slice header points to &a[1].
That’s why &c[0] prints a different address — it’s offset by one element.

So c shares the same backing array as a and b, but its slice header’s pointer field starts at index 1.
```go
import "unsafe"

fmt.Printf("c backing array base: %p\n", unsafe.Pointer(&c[:cap(c)][0]))

```

c[:cap(c)] → makes a slice covering the entire capacity of c.
&c[:cap(c)][0] → address of the first element in that full‑capacity slice.

&c[0] → pointer to where c starts (offset into the array).
unsafe.Pointer(&c[:cap(c)][0]) → pointer to the base of the backing array that c is sharing.


```go
func main() {
	a := make([]int, 0, 3)
	b := append(a, 1, 2, 3)
	fmt.Printf("a: %v, addr(slice header): %p, addr(backing array):\n", a, &a )
	fmt.Printf("b: %v, addr(slice header): %p, addr(backing array): %p-%p\n", b, &b, &b[0], &b[1])
	a = append(a, 9, 10)
	fmt.Printf("a: %v, addr(slice header): %p, addr(backing array):%p-%p\n", a, &a, &a[0], &a[1])
	fmt.Printf("b: %v, addr(slice header): %p, addr(backing array): %p-%p\n", b, &b, &b[0], &b[1])
    b = append(b, 10, 20, 30)
    fmt.Printf("a: %v, addr(slice header): %p, addr(backing array):%p-%p\n", a, &a, &a[0], &a[1])
	fmt.Printf("b: %v, addr(slice header): %p, addr(backing array): %p-%p\n", b, &b, &b[0], &b[1])
}
```

```bash
a: [], len=0, cap=3, addr(slice header): 0xc000010048
b: [1 2 3], len=3, cap=3, addr(slice header): 0xc000010060, addr(backing array): 0xc000018018-0xc000018020
a: [9 10], len=2, cap=3, addr(slice header): 0xc000010048, addr(backing array): 0xc000018018-0xc000018020
b: [9 10 3], len=3, cap=3, addr(slice header): 0xc000010060, addr(backing array): 0xc000018018-0xc000018020
a: [9 10], len=2, cap=3, addr(slice header): 0xc000010048, addr(backing array): 0xc000018018-0xc000018020
b: [9 10 3 10 20 30], len=6, cap=6, addr(slice header): 0xc000010060, addr(backing array): 0xc00010e000-0xc00010e008
```

```go
func main() {
	a := make([]int, 1, 4)
	b := append(a, 1, 2, 3)

	fmt.Printf("a: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p\n", a, len(a), cap(a), &a, &a[0])
	fmt.Printf("b: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p-%p\n", b, len(b), cap(b), &b, &b[0], &b[1])

	a = append(a, 9, 10)
	fmt.Printf("a: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p-%p\n", a, len(a), cap(a), &a, &a[0], &a[1])
	fmt.Printf("b: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p-%p\n", b, len(b), cap(b), &b, &b[0], &b[1])
	b = append(b, 10, 20, 30)
	fmt.Printf("a: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p-%p\n", a, len(a), cap(a), &a, &a[0], &a[1])
	fmt.Printf("b: %v, len=%d, cap=%d, addr(slice header): %p, addr(backing array): %p-%p\n", b, len(b), cap(b), &b, &b[0], &b[1])
}
```
```bash
a: [0], len=1, cap=4, addr(slice header): 0xc000010048, addr(backing array): 0xc000100000
b: [0 1 2 3], len=4, cap=4, addr(slice header): 0xc000010060, addr(backing array): 0xc000100000-0xc000100008
a: [0 9 10], len=3, cap=4, addr(slice header): 0xc000010048, addr(backing array): 0xc000100000-0xc000100008
b: [0 9 10 3], len=4, cap=4, addr(slice header): 0xc000010060, addr(backing array): 0xc000100000-0xc000100008
a: [0 9 10], len=3, cap=4, addr(slice header): 0xc000010048, addr(backing array): 0xc000100000-0xc000100008
b: [0 9 10 3 10 20 30], len=7, cap=8, addr(slice header): 0xc000010060, addr(backing array): 0xc00010c040-0xc00010c048
```

```go
func main() {
	a := []int{1, 2, 3}
	fmt.Println(len(a), cap(a)) // 3 3
	b := a[:2] // b is a slice of the first two elements of a
	fmt.Println(a, b) // [1 2 3] [1 2]
	fmt.Println(len(b), cap(b)) // 2 3 // So b points to [1, 2] but can still "see" the third element of a if extended.
	c := append(b, 4) // c := append([]int(nil), b...) c = append(c, 4) - This way, c gets its own backing array and won’t affect a.
	// b has length 2 and capacity 3. Appending one element (4) fits within its capacity, so Go does not allocate a new array. Instead, it overwrites the next slot in the shared backing array (the third element of a).
	fmt.Println(a, b, c)  // [1 2 4] [1 2] [1 2 4]
	fmt.Println(len(c), cap(c)) // 3 3
}
```
```go
func main() {
	a := []int{1, 2, 3, 4} 
	fmt.Println(len(a), cap(a)) // 4 4
	b := a[1:3]
	fmt.Println(a, b) // [1 2 3 4] [2 3]
	fmt.Println(len(b), cap(b)) // 2 3 // Length is simply end - start = 3 - 1 = 2
	// Capacity is how many elements you can fit starting from the slice’s first element until the end of the backing array. b starts at index 1 of a. From index 1 to the end of a (index 3) there are 3 elements: [2, 3, 4] So cap(b) = 3
}
```
```go
func main() {
	a := []int{1, 2, 3, 4}
	b := []int{0, 0}
	copy(b, a) // The rule: copy(dst, src) copies min(len(dst), len(src)) elements.
	fmt.Println(a, b) // 1 2 3 4] [1 2]
}
// copy only copies as many elements as the destination slice can hold. It never resizes the destination. That’s why b ends up with [1, 2] instead of [1, 2, 3, 4].
```

```go
func main() {
	a := []int{1, 2, 3}
	b := []int{0, 0, 0, 0, 0, 0, 0}
	copy(b, a)       
	fmt.Println(a, b) // [1 2 3] [1 2 3 0 0 0 0]
}
```
```go
func main() {
	a := []int{1, 2, 3, 4}
	b := append(a[:0], 4, 6, 8)
	fmt.Println(a, b) // [4 6 8 4] [4 6 8]
}
// This is a slice of length 0 but capacity 4 (because it starts at index 0 and can extend to the end of a).
// So it’s an empty view into the same backing array as a.
// a[:0] has capacity 4, so appending 3 elements fits without allocating a new array.
// The values 4, 6, 8 are written into the backing array starting at index 0.
// Appending to a overwrote the first three elements of a. The last element (a[3]) remained unchanged (4), so you see [4, 6, 8, 4].
```
```go
func main() {
	a := []int{1, 2, 3, 4}
	// a[:1] // → [1] (len=1, cap=4)
	// a[2:] // → [3, 4] (len=2, cap=2)
	b := append(a[:1], a[2:]...) // 
	fmt.Println(a, b) // [1 3 4 4] [1 3 4]
}
// a[:1] has length 1, capacity 4 (because it starts at index 0 and can grow to the end of a)
// Appending [3, 4] fits within that capacity (1 + 2 = 3 ≤ 4).
// So Go reuses the same backing array instead of allocating a new one.
// Since b reused a’s backing array: The append overwrote elements in a starting at index 1.
```
```go
func main() {
	a := []int{1, 2, 3, 4}
	b := a[:3]
	c := b[:4]
	fmt.Println(a, b, c) // [1 2 3 4] [1 2 3] [1 2 3 4]
}
```

```go
func main() {
	s := []int{1, 2, 3}
	for i := range s { // range s evaluates the length of s once at the beginning (which is 3). So the loop runs exactly 3 times, even though s grows during the loop.
		s = append(s, i)
	}
	fmt.Println(s)
	for i := 0; i < len(s); i++ {  // Now len(s) is re-evaluated on every iteration. Initially, len(s) = 6 (from the first loop). But each iteration appends one element, so len(s) keeps increasing. This means the loop never terminates — it’s an infinite loop.
	if i==40{
		break
	}
		s = append(s, i)
	}
}
```
```bash
thegtdev@theGTdev:~/Projects/Golangs/MASTER-DEV-INTERVIEW/Golang$ go run main.go 
[1 2 3 0 1 2]
[1 2 3 0 1 2 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39]
thegtdev@theGTdev:~/Projects/Golangs/MASTER-DEV-INTERVIEW/Golang$ 
```
```go
package main

import "fmt"

func addElement(s []int) {
    s = append(s, 100) // local header updated
    s[0]=99
    fmt.Println("Inside function:", s, "len:", len(s))
}

func main() {
    a := []int{1, 2}
    addElement(a)
    fmt.Println("In main:", a, "len:", len(a))
}
// Inside function: [99 2 100] len: 3
// In main: [1 2] len: 2

```

```go
package main

import "fmt"

func addElement(s []int) []int {
    s = append(s, 100) // local header updated
    return s           // return new header
}

func main() {
    a := []int{1, 2}
    fmt.Printf("a: %v, addr(slice header): %p, addr(backing array): %p-%p\n", a, &a, &a[0], &a[1])
    a = addElement(a) // assign returned slice
    fmt.Println("In main:", a, "len:", len(a))
    fmt.Printf("a: %v, addr(slice header): %p, addr(backing array): %p-%p\n", a, &a, &a[0], &a[1])
}
/*
Now the updated slice header is returned and assigned back to a.
Caller sees the new length and can access the appended element.

a: [1 2], addr(slice header): 0xc000010048, addr(backing array): 0xc000012050-0xc000012058
In main: [1 2 100] len: 3
a: [1 2 100], addr(slice header): 0xc000010048, addr(backing array): 0xc000102020-0xc000102028
*/
```
