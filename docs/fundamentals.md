## Go Fundamentals (Types, Structs, Interfaces, Functions, Generics)

This cheatsheet covers fundamental Go language features excluding concurrency primitives.

### 1. Basic & Composite Types

*   **Basic Types:**
    *   `bool`
    *   `string` (immutable sequence of bytes, usually UTF-8)
    *   Integer types: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8` (alias `byte`), `uint16`, `uint32`, `uint64`, `uintptr`
    *   Floating-point types: `float32`, `float64`
    *   Complex types: `complex64`, `complex128`
*   **Type Declarations:** Create new named types.
    ```go
    type UserID int
    type Point struct { X, Y float64 }
    type HandlerFunc func(http.ResponseWriter, *http.Request)
    ```
*   **Type Alias:** Create an alias (alternative name) for an existing type. *Rarely needed*.
    ```go
    type B = byte // B is just another name for byte
    type UserMap = map[string]User 
    ```
*   **Zero Values:** Variables declared without an explicit initial value are given their zero value:
    *   `0` for numeric types
    *   `false` for `bool`
    *   `""` (empty string) for `string`
    *   `nil` for pointers, functions, interfaces, slices, channels, and maps.
*   **Type Conversion:** Explicit conversion is required between different types.
    ```go
    var i int = 10
    var f float64 = float64(i) // Explicit conversion
    // var f float64 = i // Compile-time error
    
    var uid UserID = 123
    var baseInt int = int(uid) // Conversion needed even if underlying type is int
    ```

### 2. Structs

Collections of named fields. Go's way of grouping data (like classes without methods directly inside).

*   **Definition:**
    ```go
    type Rectangle struct {
        Width  float64
        Height float64
        Label  string // Exported field (starts with uppercase)
        area   float64 // Unexported field (starts with lowercase)
    }
    ```
*   **Instantiation:**
    ```go
    // Zero-valued struct
    var r1 Rectangle 
    
    // Struct literal (order matters if field names omitted)
    r2 := Rectangle{10.0, 5.0, "Rect A", 0} 
    
    // Struct literal with field names (order doesn't matter, preferred)
    r3 := Rectangle{Width: 10.0, Height: 5.0, Label: "Rect B"} 
    
    // Using new() - returns a pointer to a zero-valued struct
    r4Ptr := new(Rectangle) // r4Ptr is *Rectangle
    
    // Pointer using struct literal
    r5Ptr := &Rectangle{Width: 20, Height: 7} 
    ```
*   **Field Access:** Use the dot (`.`) operator. Works on both struct values and pointers.
    ```go
    r3.Width = 11.0
    r5Ptr.Height = 8.0 // Go automatically dereferences pointers for field access
    ```
*   **Embedding (Composition):** Include a struct type directly without a field name. Promotes methods and fields of the embedded type. *Go's alternative to inheritance.*
    ```go
    type ColoredRectangle struct {
        Rectangle // Embedded type
        Color     string
    }
    
    cr := ColoredRectangle{
        Rectangle: Rectangle{Width: 5, Height: 3},
        Color:     "Blue",
    }
    
    fmt.Println(cr.Width) // Access embedded field directly
    cr.Width = 6          // Modify embedded field directly
    ```

### 3. Collections: Arrays, Slices, Maps

*   **Arrays:** Fixed-size sequence of elements of the same type. *Less common than slices.*
    ```go
    var a [3]int // Array of 3 integers, initialized to zero values {0, 0, 0}
    b := [3]int{1, 2, 3}
    c := [...]int{1, 2, 3, 4} // Compiler counts elements
    
    fmt.Println(len(a)) // Length is part of the type
    val := a[0]         // Access element
    a[0] = 10           // Modify element
    ```
*   **Slices:** Dynamically-sized, flexible view into the elements of an *underlying array*. More common and powerful than arrays.
    ```go
    // Create using make(type, length, capacity?)
    s1 := make([]string, 5)      // len=5, cap=5
    s2 := make([]int, 0, 10)    // len=0, cap=10
    
    // Create using a slice literal
    s3 := []bool{true, false, true} // len=3, cap=3
    
    // Create by slicing an existing array or slice
    arr := [5]int{1, 2, 3, 4, 5}
    s4 := arr[1:4] // Includes elements at index 1, 2, 3. len=3, cap=4 (from index 1 to end of arr)
    
    // Length and Capacity
    fmt.Println(len(s4), cap(s4))
    
    // Append: Returns a NEW slice (underlying array might change!)
    s4 = append(s4, 6) // s4 might now point to a different underlying array if cap was exceeded
    fmt.Println(s4) // {2, 3, 4, 6}
    
    // Nil slice: Has len 0, cap 0, no underlying array.
    var nilSlice []int
    ```
*   **Maps:** Unordered collection of key-value pairs. Keys must be comparable types.
    ```go
    // Create using make
    m1 := make(map[string]int)
    
    // Create using a map literal
    m2 := map[string]string{
        "name": "Alice",
        "city": "New York",
    }
    
    // Add/Update elements
    m1["count"] = 1
    m1["count"]++
    
    // Retrieve elements (use two-value assignment to check existence)
    val, ok := m2["city"]
    if ok {
        fmt.Println("City:", val)
    } else {
        fmt.Println("City not found")
    }
    
    // Delete elements
    delete(m2, "city")
    
    // Iterate (order is not guaranteed)
    for key, value := range m2 {
        fmt.Printf("%s: %s\n", key, value)
    }
    
    // Nil map: Cannot add elements to a nil map (runtime panic)
    var nilMap map[string]int 
    // nilMap["key"] = 1 // PANIC! Must initialize first: nilMap = make(map[string]int)
    ```

* **Built-ins (Go 1.21):** New general language features:
    *   `min(x, y, ...)`: Returns the smallest comparable value.
    *   `max(x, y, ...)`: Returns the largest comparable value.
    *   `clear(m)`: Deletes all entries from map `m`.
    *   `clear(s)`: Zeroes elements in slice `s` (length/capacity remain).
    ```go
    smallest := min(5, 2, 9) // smallest is 2
    largest := max(3.1, 4.5, 1.2) // largest is 4.5
    
    myMap := map[string]int{"a": 1, "b": 2}
    clear(myMap) // myMap is now empty: map[]
    
    mySlice := []int{10, 20, 30}
    clear(mySlice) // mySlice is now {0, 0, 0}
    ```
* **`slices` and `maps` packages:** Use the standard library `slices` and `maps` packages (stabilized around Go 1.21) for common operations, e.g. functions like `slices.Sort`, `slices.Compact`, `slices.Delete`, `maps.Clone`, `maps.Equal`, etc., as alternatives to manual implementations.

### 4. Functions & Methods

*   **Functions:** Basic building blocks.
    ```go
    func add(a int, b int) int { // Parameters and return type
        return a + b
    }
    
    func swap(a, b string) (string, string) { // Multiple return values
        return b, a
    }
    
    func divide(a, b float64) (result float64, err error) { // Named return values
        if b == 0 {
            err = errors.New("division by zero")
            return // Implicitly returns the current values of result (0.0) and err
        }
        result = a / b
        return // Implicitly returns result and err (nil)
        // NOTE: Use named returns sparingly, can reduce clarity.
    }
    
    func sum(nums ...int) int { // Variadic function (treats nums as []int)
        total := 0
        for _, num := range nums {
            total += num
        }
        return total
    }
    
    // Function call
    sumResult := sum(1, 2, 3, 4) 
    ```
*   **Closures:** Functions can access variables from their surrounding scope (lexical scoping).
    ```go
    func makeIncrementor() func() int {
        i := 0
        return func() int { // This inner function is a closure
            i++
            return i
        }
    }
    
    inc := makeIncrementor()
    fmt.Println(inc()) // 1
    fmt.Println(inc()) // 2
    ```
*   **`defer` Statement:** Schedules a function call to be run just before the function containing the `defer` returns. Useful for cleanup (e.g., closing files, unlocking mutexes). Deferred calls are executed in LIFO (Last-In, First-Out) order.
    ```go
    func processFile(filename string) error {
        f, err := os.Open(filename)
        if err != nil { return err }
        defer f.Close() // Guarantees f.Close() is called before processFile returns

        // ... use f ...
        return nil 
    }
    ```
*   **Methods:** Functions with a special receiver argument, associated with a specific type.
    ```go
    type Circle struct { Radius float64 }
    
    // Method with a value receiver (operates on a copy)
    func (c Circle) Area() float64 {
        // c.Radius = 10 // Modifying c here won't affect the original Circle
        return math.Pi * c.Radius * c.Radius
    }
    
    // Method with a pointer receiver (operates on the original value)
    func (c *Circle) Scale(factor float64) {
        c.Radius *= factor // Modifies the original Circle
    }
    
    // Usage:
    circ := Circle{Radius: 5}
    fmt.Println(circ.Area()) // Call method on value
    
    circPtr := &Circle{Radius: 10}
    circPtr.Scale(2) // Call method on pointer
    fmt.Println(circPtr.Radius) // 20
    
    // Go automatically handles value/pointer access for method calls:
    circ.Scale(0.5) // Valid: Go implicitly takes (&circ).Scale(0.5)
    fmt.Println(circPtr.Area()) // Valid: Go implicitly takes (*circPtr).Area()
    ```
    *   **Choose Pointer vs Value Receiver:**
        *   Use a **pointer receiver** if the method needs to modify the receiver.
        *   Use a **pointer receiver** if the struct is large (avoids copying).
        *   Use a **value receiver** if the method doesn't modify the receiver and the struct is small, or if you need immutability guarantees for the receiver within the method. Consistency across methods for a given type is important.

### 5. Interfaces

Define behavior by specifying a set of method signatures. A type *implicitly* satisfies an interface if it implements all methods declared in the interface.

*   **Definition:**
    ```go
    type Shape interface {
        Area() float64
    }
    
    type Object interface {
        Volume() float64
    }
    
    // Interface embedding
    type MaterialObject interface {
        Shape
        Object
        Material() string
    }
    ```
*   **Implementation:** Implicit - no `implements` keyword.
    ```go
    // Rectangle already has Area() method via pointer receiver *if defined above*
    // func (r *Rectangle) Area() float64 { ... }
    
    // Circle has Area() method via value receiver
    // func (c Circle) Area() float64 { ... }
    
    func PrintArea(s Shape) { // Function accepts any type that satisfies the Shape interface
        fmt.Println("Area:", s.Area())
    }
    
    // Usage:
    r := &Rectangle{Width: 10, Height: 4} // Use pointer if Area() has pointer receiver
    c := Circle{Radius: 5}
    
    PrintArea(r)
    PrintArea(c) 
    ```
*   **Empty Interface (`interface{}` or `any`):** Specifies zero methods. All types satisfy the empty interface. Use sparingly as it loses static type safety. `any` is an alias for `interface{}` introduced in Go 1.18.
    ```go
    var i interface{} // or var i any
    i = 42
    i = "hello"
    ```
*   **Type Assertions:** Access the underlying concrete value from an interface value.
    ```go
    var val any = "hello world"
    
    s, ok := val.(string) // Recommended: Check if assertion is valid
    if ok {
        fmt.Printf("String value: %s\n", s)
    } else {
        fmt.Println("Not a string")
    }
    
    // s = val.(string) // Panics if val is not a string
    ```
*   **Type Switches:** Perform different actions based on the underlying concrete type of an interface variable.
    ```go
    func checkType(v any) {
        switch t := v.(type) {
        case string:
            fmt.Println("It's a string:", t)
        case int:
            fmt.Println("It's an int:", t)
        case bool:
            fmt.Println("It's a bool:", t)
        case nil:
            fmt.Println("It's nil")
        default:
            fmt.Printf("Unknown type: %T\n", t) // %T prints the type
        }
    }
    ```

### 6. Generics (Go 1.18+)

Write code that works with multiple types without using `interface{}`/`any`.

*   **Type Parameters:** Declared in square brackets `[]` after the function/type name.
    ```go
    // Generic function to find the minimum of two orderable values
    func Min[T constraints.Ordered](a, b T) T { // constraints.Ordered includes ~, <, <=, etc.
        if a < b {
            return a
        }
        return b
    }
    
    // Using the generic function (type argument often inferred)
    minInt := Min(5, 3)       // T inferred as int
    minFloat := Min(3.14, 2.71) // T inferred as float64
    minString := Min("apple", "banana") // T inferred as string
    // minComplex := Min(complex(1,1), complex(2,2)) // Error: complex not in constraints.Ordered
    ```
*   **Type Constraints:** Define what methods or properties a type parameter must satisfy. Can use interfaces or predefined constraints (like `comparable`, `constraints.Ordered`, `constraints.Integer`, etc.).
    ```go
    // Custom interface constraint
    type Stringer interface {
        String() string
    }
    
    func PrintString[T Stringer](val T) {
        fmt.Println(val.String())
    }
    
    // Using `comparable` built-in constraint
    func AreEqual[T comparable](a, b T) bool {
        return a == b
    }
    ```
*   **Generic Types:** Structs, interfaces, etc., can also have type parameters.
    ```go
    type Stack[T any] struct { // Generic stack for any type
        items []T
    }
    
    func (s *Stack[T]) Push(item T) {
        s.items = append(s.items, item)
    }
    
    func (s *Stack[T]) Pop() (T, bool) {
        if len(s.items) == 0 {
            var zero T // Get zero value for type T
            return zero, false
        }
        item := s.items[len(s.items)-1]
        s.items = s.items[:len(s.items)-1]
        return item, true
    }
    
    // Instantiation
    intStack := Stack[int]{}
    intStack.Push(10)
    intStack.Push(20)
    val, _ := intStack.Pop() // val is 20 (type int)
    
    stringStack := Stack[string]{}
    stringStack.Push("hello")
    ```

### 7. Error Handling

Go uses an explicit error-checking convention via the built-in `error` interface.

*   **`error` Interface:**
    ```go
    type error interface {
        Error() string
    }
    ```
*   **Convention:** Functions that can fail return an `error` as their last return value. A `nil` error indicates success.
    ```go
    import "errors"
    import "fmt"
    
    func process(input int) (string, error) {
        if input < 0 {
            // Create simple errors
            return "", errors.New("input cannot be negative") 
        }
        if input == 0 {
             // Create formatted errors
            return "", fmt.Errorf("input cannot be zero (value was %d)", input)
        }
        return fmt.Sprintf("Processed: %d", input), nil // Success: return result and nil error
    }
    
    // Checking for errors
    result, err := process(-5)
    if err != nil {
        fmt.Println("Error occurred:", err)
        // Handle error (e.g., return, log, retry)
    } else {
        fmt.Println("Success:", result)
    }
    ```
* `errors.Join` (Go 1.20): provides a standard way to combine multiple errors, common when processing multiple items concurrently or sequentially where multiple failures can occur.
    ```go
    err1 := errors.New("first error")
    err2 := fmt.Errorf("second error with details: %w", io.EOF)

    joinedErr := errors.Join(err1, nil, err2) // nil errors are ignored

    fmt.Println(joinedErr) // Output: first error\nsecond error with details: EOF

    // Can still use errors.Is/As on the joined error
    if errors.Is(joinedErr, io.EOF) {
        fmt.Println("Joined error contains EOF")
    }
    ```

---

This cheatsheet covers the non-concurrent fundamentals. Remember Go's emphasis on simplicity, readability, and explicitness.
