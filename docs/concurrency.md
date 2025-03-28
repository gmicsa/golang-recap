## Go Concurrency

This cheatsheet covers the fundamentals and common patterns for writing concurrent Go programs.

### 1. Goroutines

The basic unit of concurrency in Go. Lightweight, independently executing functions managed by the Go runtime.

*   **Starting a Goroutine:**
    ```go
    go funcName(args) // Start a named function
    
    go func(params) { // Start an anonymous function (closure)
        // ... function body ...
    }(args) 
    ```
*   **Key Characteristics:**
    *   Lightweight (stack starts small, grows as needed).
    *   Multiplexed onto OS threads (M:N scheduling).
    *   Cheap to create (thousands or millions are feasible).
*   **`main` Goroutine:** The program exits when the `main` goroutine finishes, even if other goroutines are still running. Use synchronization (like `sync.WaitGroup`) to wait for others if needed.

### 2. Channels

Typed conduits for communication and synchronization *between* goroutines. "Share memory by communicating."

*   **Creating Channels:**
    ```go
    // Unbuffered channel: Send blocks until Receive, Receive blocks until Send
    ch := make(chan int) 
    
    // Buffered channel: Send blocks only if buffer is full, Receive blocks only if buffer is empty
    bufCh := make(chan string, 10) // Buffer size 10
    ```
*   **Sending & Receiving:**
    ```go
    ch <- value    // Send value to channel ch (blocks if unbuffered or buffer full)
    value := <-ch  // Receive value from channel ch (blocks if empty)
    
    // Non-blocking receive (check if channel open and value available)
    value, ok := <-ch 
    // ok == true: value received successfully
    // ok == false: channel is closed and empty
    ```
*   **Closing Channels:**
    ```go
    close(ch)
    ```
    *   **Who closes?** The *sender* should close the channel to signal no more values will be sent.
    *   **Never close a channel multiple times** (panic).
    *   **Never close a `nil` channel** (panic).
    *   Receiving from a closed channel returns the zero value for the type immediately (`value, ok := <-ch` gives `zeroValue, false`).
    *   Sending to a closed channel causes a panic.
*   **Ranging over Channels:**
    ```go
    // Loops until the channel is closed and drained
    for item := range ch {
        fmt.Println("Received:", item)
    } 
    ```
*   **Directional Channels:** Used in function signatures to enforce channel usage.
    ```go
    func sender(ch chan<- int) { // Can ONLY send to ch
        ch <- 1
        // x := <-ch // Compile-time error
    }
    
    func receiver(ch <-chan int) { // Can ONLY receive from ch
        fmt.Println(<-ch)
        // ch <- 2 // Compile-time error
    }
    ```
*   **`select` Statement:** Waits on multiple channel operations simultaneously.
    ```go
    select {
    case msg1 := <-ch1:
        fmt.Println("Received from ch1:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Received from ch2:", msg2)
    case ch3 <- value:
        fmt.Println("Sent to ch3")
    case <-time.After(1 * time.Second): // Timeout
        fmt.Println("Timeout occurred")
    // default: // Optional: Makes the select non-blocking
    //     fmt.Println("No communication ready")
    }
    ```
    *   Blocks until one case can proceed.
    *   If multiple cases are ready, one is chosen pseudo-randomly.
    *   `default` case runs if no other case is ready immediately.

### 3. `sync` Package

Provides basic synchronization primitives for shared memory access (use cautiously, prefer channels when possible).

*   **`sync.Mutex` (Mutual Exclusion Lock):** Protects shared data from simultaneous access.
    ```go
    var mu sync.Mutex
    var counter int
    
    // In a goroutine:
    mu.Lock()
    counter++ 
    // Critical section accessing shared data
    mu.Unlock() 
    
    // Idiomatic usage with defer:
    mu.Lock()
    defer mu.Unlock() // Unlock happens when the function returns
    counter++ 
    ```
*   **`sync.RWMutex` (Reader/Writer Lock):** Allows multiple readers OR one writer. Good for read-heavy workloads.
    ```go
    var rwMu sync.RWMutex
    var data map[string]string
    
    // Reader Goroutine:
    rwMu.RLock()
    value := data["key"]
    rwMu.RUnlock()
    
    // Writer Goroutine:
    rwMu.Lock()
    data["key"] = "new_value"
    rwMu.Unlock()
    
    // Use defer idiomatically as with Mutex (defer rwMu.RUnlock() / defer rwMu.Unlock())
    ```
*   **`sync.WaitGroup`:** Waits for a collection of goroutines to finish.
    ```go
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1) // Increment counter (before starting goroutine)
        go func(id int) {
            defer wg.Done() // Decrement counter when goroutine finishes
            fmt.Println("Worker", id, "done")
            time.Sleep(100 * time.Millisecond)
        }(i)
    }
    
    wg.Wait() // Block until counter is zero
    fmt.Println("All workers finished")
    ```
    *   `Add()` *before* starting the goroutine.
    *   `Done()` typically via `defer` inside the goroutine.
*   **`sync.Once`:** Executes an action exactly once. Useful for initialization.
    ```go
    var once sync.Once
    var config *Config
    
    func GetConfig() *Config {
        once.Do(func() {
            fmt.Println("Initializing config...")
            config = &Config{ /* load config */ } 
        })
        return config
    }
    ```
*   **`sync.Pool`:** A pool of temporary objects that can be reused. Reduces allocation pressure.
    ```go
    var bufferPool = sync.Pool{
        New: func() interface{} { // Called when Get() finds no item
            return new(bytes.Buffer)
        },
    }
    
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset() // Important: Reset state before use
    // ... use buf ...
    bufferPool.Put(buf) // Put back into the pool for reuse
    ```
    *   Objects in the pool can be garbage collected at any time. Not for persistent state.
*   **`sync.Cond`:** A condition variable for goroutines waiting for or announcing an event. Less common than channels or WaitGroup, used for more complex signaling.

### 4. Common Concurrency Patterns

> Note on loop variable capture semantics introduced by default in Go 1.22

* Old Pitfall (Pre-1.22):

    ```go
    // Pre-Go 1.22 common mistake
    for _, v := range values {
        go func() {
            fmt.Println(v) // All goroutines likely print the *last* value of v!
        }()
    }
    // Pre-Go 1.22 Fix:
    for _, v := range values {
        v := v // Create a new variable scoped to the loop iteration
        go func() {
            fmt.Println(v)
        }()
    }
    ```
    
*  New Behavior (Go 1.22+):

    ```go
    // Go 1.22+ Behavior (Default)
    for _, v := range values {
        go func() {
            fmt.Println(v) // Each goroutine captures the value of v for *that specific iteration*.
        }()
    }
    ```

*   **Worker Pool:** Limit concurrency, reuse resources.
    ```go
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    numWorkers := 5
    var wg sync.WaitGroup
    
    // Start workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go func(id int, jobs <-chan int, results chan<- int) {
            defer wg.Done()
            for j := range jobs { // Reads until jobs is closed
                fmt.Printf("Worker %d started job %d\n", id, j)
                time.Sleep(time.Millisecond * 500) // Simulate work
                results <- j * 2
                fmt.Printf("Worker %d finished job %d\n", id, j)
            }
        }(w, jobs, results)
    }
    
    // Send jobs
    for j := 1; j <= 10; j++ {
        jobs <- j
    }
    close(jobs) // Close jobs channel to signal workers
    
    // Wait for all workers to finish processing jobs
    wg.Wait() 
    close(results) // Close results *after* all workers are done
    
    // Collect results (optional)
    for r := range results {
        fmt.Println("Result:", r)
    }
    ```
*   **Fan-Out, Fan-In:** Distribute work (Fan-Out), Collect results (Fan-In).
    *   **Fan-Out:** One goroutine sends tasks to multiple worker goroutines via channels.
    *   **Fan-In:** Multiple goroutines send results to a single channel. Often requires a `WaitGroup` to know when all inputs are done before closing the output channel.
*   **Rate Limiting:** Control the frequency of operations.
    ```go
    // Using time.Ticker
    rateLimiter := time.NewTicker(200 * time.Millisecond) // Allow 5 operations per second
    defer rateLimiter.Stop()
    
    for req := range requests {
        <-rateLimiter.C // Block until next tick
        go handle(req)
    }
    
    // Using a buffered channel (token bucket)
    tokenBucket := make(chan struct{}, 5) // Capacity 5
    // Fill initially or periodically
    for i := 0; i < 5; i++ { tokenBucket <- struct{}{} } 
    
    for req := range requests {
         <-tokenBucket // Acquire a token (blocks if empty)
         go func(req Request) {
             handle(req)
             tokenBucket <- struct{}{} // Release token back
         }(req)
    }
    ```
*   **Cancellation / Timeout (`context.Context`):** Propagate cancellation signals or deadlines. **Crucial for robust concurrent code.**
    ```go
    func longOperation(ctx context.Context, resultChan chan<- string) {
        select {
        case <-time.After(5 * time.Second): // Simulate work
            resultChan <- "Operation completed"
        case <-ctx.Done(): // Check if context was cancelled
            fmt.Println("Operation cancelled:", ctx.Err())
            // Clean up resources if necessary
            return 
        }
    }
    
    func main() {
        // Create a context with a 3-second timeout
        ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        defer cancel() // Important: Always call cancel to release resources
    
        resultChan := make(chan string, 1)
        go longOperation(ctx, resultChan)
    
        select {
        case res := <-resultChan:
            fmt.Println(res)
        case <-ctx.Done(): // This will trigger if the timeout expires before longOperation finishes
            fmt.Println("Main detected cancellation/timeout:", ctx.Err())
        }
    }
    ```

### 5. Pitfalls & Best Practices

*   **Race Conditions:** Multiple goroutines accessing shared data concurrently, at least one access is a write.
    *   **Detection:** Use the race detector: `go run -race main.go`, `go test -race ./...`
    *   **Prevention:** Use channels for communication or `sync` primitives (`Mutex`, `RWMutex`) for synchronization.
*   **Deadlocks:** Goroutines waiting for each other indefinitely.
    *   Common causes: Unbuffered channel sends/receives without a corresponding receiver/sender; incorrect mutex locking order (A locks M1 then M2, B locks M2 then M1).
    *   Prevention: Careful design, consistent lock ordering, use `select` with timeouts.
*   **Goroutine Leaks:** Goroutines that block indefinitely and are never cleaned up.
    *   Common causes: Sending on a channel with no receiver, receiving from a channel that's never closed or sent to, blocked mutexes.
    *   Prevention: Use `context.Context` for cancellation, ensure channels are closed properly, use `WaitGroup` correctly.
*   **Channel Closing:** Remember the rules: Sender closes, don't close multiple times, check `ok` on receive if needed. Ranging handles closed channels gracefully.
*   **Prefer Channels over Mutexes:** When designing communication between goroutines, channels often lead to clearer, less error-prone code. Use mutexes primarily to protect the internal state *within* a single logical component accessed by multiple goroutines.
*   **Use `context.Context`:** For cancellation, deadlines, and passing request-scoped values. It's the standard way to manage goroutine lifecycles in libraries and applications.
*   **Keep Critical Sections Short:** Minimize the time spent holding locks (`sync.Mutex`) to reduce contention.

---

This cheatsheet provides a solid foundation. Remember to test thoroughly, especially with the `-race` flag enabled! Happy Go-ing!
