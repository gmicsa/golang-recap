## Go Error Handling

This cheatsheet covers standard practices, patterns, and nuances for handling errors effectively in Go.

### 1. The `error` Interface

The foundation of Go error handling. It's a simple, built-in interface.

```go
type error interface {
    Error() string
}
```

*   Any type that implements the `Error() string` method satisfies the `error` interface.
*   A `nil` error value indicates success. A non-nil value indicates failure.

### 2. Basic Error Handling Convention

Functions that can fail typically return an `error` as their last return value.

*   **Function Signature:** `func DoSomething(...) (ResultType, error)` or `func DoSomethingElse(...) error`
*   **Checking for Errors:** Always check the returned error immediately.
    ```go
    result, err := DoSomething(input)
    if err != nil {
        // Handle the error (log, return, fallback, etc.)
        log.Printf("Error doing something with %v: %v", input, err)
        return defaultValue, err // Often propagate the error up
    }
    // Success: proceed with 'result'
    fmt.Println("Success:", result)
    ```
*   **Never ignore errors:** `_, _ = DoSomething()` is bad practice unless you explicitly intend to ignore the result *and* the error (very rare and dangerous).

### 3. Creating Errors

*   **`errors.New(text string)`:** Creates a simple error with a static message. Use for errors where the text is constant.
    ```go
    import "errors"

    var ErrInvalidInput = errors.New("invalid input provided") 

    func CheckInput(val int) error {
        if val < 0 {
            return ErrInvalidInput // Return the predefined error
        }
        return nil
    }
    ```
*   **`fmt.Errorf(format string, args ...any)`:** Creates an error with a formatted message. Use when the error message needs dynamic content.
    ```go
    import "fmt"

    func ProcessFile(filename string) error {
        // ... try to open file ...
        if err != nil {
             // Create a new error with context
            return fmt.Errorf("failed to open file '%s': %v", filename, err) 
        }
        // ...
        return nil
    }
    ```

### 4. Wrapping Errors (Go 1.13+)

Crucial for adding context without obscuring the original error cause.

*   **`fmt.Errorf` with `%w` verb:** Wraps the original error. The resulting error holds a reference to the underlying error.
    ```go
    func ReadConfig() error {
        err := LoadFile("config.yaml")
        if err != nil {
            // Wrap the error from LoadFile
            return fmt.Errorf("error reading configuration: %w", err) 
        }
        // ...
        return nil
    }
    ```
*   **`errors.Unwrap(err error)`:** Retrieves the underlying error wrapped by `err` (if any). Returns `nil` if `err` does not wrap another error.
    ```go
    err := ReadConfig()
    if err != nil {
        originalErr := errors.Unwrap(err)
        if originalErr != nil {
            fmt.Println("Original cause:", originalErr) // Might print "file not found" etc.
        }
    }
    // Can be used in a loop to unwrap multiple layers
    for err != nil {
        fmt.Println(err)
        err = errors.Unwrap(err)
    }
    ```

### 5. Checking Errors

How to determine the *nature* of an error.

*   **Checking for `nil`:** The most basic check for success/failure.
*   **`errors.Is(err, target error)`:** Checks if `err` (or any error in its wrapped chain) *is* equivalent to the `target` error. **Preferred way to check for specific sentinel error values.**
    ```go
    import "os"

    content, err := os.ReadFile("nonexistent.txt")
    if err != nil {
        if errors.Is(err, os.ErrNotExist) { // Checks if err or its wrapped cause is os.ErrNotExist
            fmt.Println("File does not exist, proceeding with defaults.")
            // Handle specific case: file not found
        } else {
            // Handle other unexpected errors
            log.Fatalf("Unexpected file error: %v", err)
        }
    }
    ```
*   **`errors.As(err error, target any)`:** Checks if `err` (or any error in its wrapped chain) *is* of a specific type. If it finds a match, it sets `target` (which must be a pointer to the error type) and returns `true`. **Preferred way to check for specific error types and access their fields.**
    ```go
    type NetworkError struct {
        Code    int
        Message string
        Err     error // Underlying error
    }
    
    func (e *NetworkError) Error() string {
        return fmt.Sprintf("network error %d: %s", e.Code, e.Message)
    }

    func (e *NetworkError) Unwrap() error { // Implement Unwrap if wrapping
        return e.Err
    }
    
    // --- Somewhere else ---
    err := MakeNetworkRequest() 
    var netErr *NetworkError // Target must be a pointer to the error type
    if errors.As(err, &netErr) { // Checks if err or its cause is a *NetworkError
        fmt.Printf("Network failure detected (Code: %d)\n", netErr.Code)
        if netErr.Code == 503 {
            // Handle Service Unavailable
        }
    } else {
        // Handle non-network errors
    }
    ```
*   **Simple Equality (`==`):** Only use for comparing against sentinel errors *if you are sure the error isn't wrapped*. `errors.Is` is generally safer.
*   **Type Assertion (`err.(*MyError)`):** Only use if you are sure the error isn't wrapped *and* you need the specific type. `errors.As` is generally safer and more idiomatic.

### 6. Sentinel Errors vs. Custom Error Types

*   **Sentinel Errors:** Predefined, exported error variables (like `io.EOF`, `os.ErrNotExist`, `sql.ErrNoRows`).
    *   Use `errors.New` to define them.
    *   Check using `errors.Is`.
    *   Good for signaling fixed, well-known conditions.
    *   Example: `var ErrUserNotFound = errors.New("user not found")`
*   **Custom Error Types:** Structs that implement the `error` interface.
    *   Useful when you need to carry additional context/data with the error (e.g., status codes, internal state).
    *   Check using `errors.As`.
    *   Implement `Unwrap()` if the custom type wraps another error.

### 7. Panic & Recover

**`panic` is NOT for normal error handling.** It's for indicating *unrecoverable* states or programmer bugs (e.g., index out of bounds, nil pointer dereference that shouldn't happen).

*   **`panic(v any)`:** Stops the ordinary flow of control, begins panicking. Execution moves up the call stack, running deferred functions.
*   **`recover()`:** Regains control of a panicking goroutine. Only useful inside `defer` functions. Returns the value passed to `panic`, or `nil` if the goroutine is not panicking.
    ```go
    func SafeExecutor(work func()) (err error) {
        defer func() {
            if r := recover(); r != nil {
                // Convert panic back to an error at the boundary
                log.Printf("Recovered from panic: %v\nStack trace:\n%s", r, debug.Stack())
                err = fmt.Errorf("internal panic: %v", r)
            }
        }()

        work() // Call the function that might panic
        return nil
    }
    ```
*   Use `recover` sparingly, typically at the boundaries of goroutines or public APIs to prevent a single failure from crashing the entire program. Don't use it to simulate exceptions like in other languages.

### 8. Best Practices & Pitfalls

*   **Handle Errors Explicitly:** Don't ignore them using `_`. Address every non-nil error.
*   **Provide Context:** Wrap errors (`%w`) or add informative details (`fmt.Errorf`) when propagating errors upwards. A raw `file not found` error deep in the stack isn't helpful without knowing *which* file or *why* it was being accessed.
*   **Return Errors, Don't Panic:** Use the standard `error` return value for expected failures (network issues, invalid input, file not found).
*   **Use `errors.Is` and `errors.As`:** Prefer these over `==` or type assertions for checking errors, especially when wrapping is involved.
*   **Define Sentinel Errors Judiciously:** Use them for stable, well-defined error conditions that callers might need to check specifically. Export them if they are part of your public API.
*   **Export Errors Carefully:** Decide if custom error types or sentinel errors need to be part of your package's public API. If so, document them clearly. If not, keep them internal.
*   **Consistency:** Maintain a consistent error handling strategy within your project/team.
*   **Clear Error Messages:** Write messages that are understandable to humans and potentially parsable by machines (if needed).
*   **Context Cancellation:** Remember that `context.Canceled` and `context.DeadlineExceeded` (available via `ctx.Err()`) are also important error conditions in concurrent/networked applications.

---

By following these patterns, you can write more robust, maintainable, and debuggable Go applications.
