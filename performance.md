## Go Performance & Optimization review

This cheatsheet covers key areas related to Go program performance, including memory management, garbage collection, profiling, benchmarking, and common optimization strategies.

### 1. Memory Management & Allocation

*   **Stack vs. Heap:**
    *   **Stack:** Used for function call frames, local variables of fixed size known at compile time. Allocation/deallocation is very fast (pointer manipulation). Managed per-goroutine. Stacks start small (e.g., 2KB) and grow/shrink as needed.
    *   **Heap:** Used for variables that *escape* their function scope (shared between goroutines, returned by pointer, stored in global variables, size unknown at compile time, interface values, closures capturing variables). Managed by the Garbage Collector. Allocation is more expensive than stack allocation.
*   **Escape Analysis:** The compiler determines if a variable can be safely allocated on the stack or if it must "escape" to the heap.
    *   **Check Escape Analysis:** `go build -gcflags="-m" ./...` (or `go run -gcflags="-m" main.go`). Look for output like `moved to heap: x` or `y escapes to heap`.
    *   **Minimize Escapes:** Prefer passing large structs by pointer, but be mindful this can sometimes *cause* escapes if the pointer itself escapes. Understand why variables escape to reduce unnecessary heap allocations.
*   **`make` vs `new`:**
    *   `make([]T, len, cap)`, `make(map[K]V, size)`, `make(chan T, buffer)`: Initializes slices, maps, and channels. Returns an initialized (not zeroed) value of type T (not *T). Allocates underlying data structures (potentially on the heap).
    *   `new(T)`: Allocates memory for a variable of type `T`, zeros the memory, and returns a pointer (`*T`). Less common than `make` for built-in collections.
*   **Slices & Maps Pre-allocation:**
    *   If you know the approximate size, pre-allocate using `make` with capacity/size hints. This avoids multiple re-allocations and copying as the slice/map grows.
    ```go
    // Avoid this if size is roughly known:
    // var data []MyStruct
    // for i := 0; i < n; i++ { data = append(data, item) } // Potential reallocations

    // Better:
    data := make([]MyStruct, 0, n) // len 0, cap n
    for i := 0; i < n; i++ { data = append(data, item) } // Usually no reallocations

    // Or if creating directly:
    data := make([]MyStruct, n) // len n, cap n
    for i := 0; i < n; i++ { data[i] = item }
    ```

### 2. Garbage Collection (GC)

Go uses a concurrent, tri-color, mark-and-sweep garbage collector aimed at low latency.

*   **Goal:** Minimize Stop-The-World (STW) pauses. Most marking and sweeping happens concurrently with the application goroutines. Short STW pauses are still needed (e.g., to enable write barrier, scan roots).
*   **Triggering:** Paced by allocation. The GC aims to start a cycle when the heap size reaches a target percentage increase over the live heap size after the *previous* GC cycle. This target is controlled by `GOGC`.
*   **`GOGC` Environment Variable:**
    *   `GOGC=100` (Default): Start GC when heap size doubles compared to live heap after the last GC.
    *   `GOGC=50`: Start GC when heap size grows 50%. More frequent GC, uses less peak memory, higher CPU cost.
    *   `GOGC=200`: Start GC when heap size grows 200%. Less frequent GC, uses more peak memory, lower CPU cost.
    *   `GOGC=off`: Disables GC. Only for specific debugging scenarios.
    *   Can be tuned via `debug.SetGCPercent()` at runtime.
*   **Write Barrier:** A small amount of code run by the compiler whenever a pointer is written to memory during the mark phase. It ensures the GC doesn't miss objects that become reachable concurrently. Has a small performance overhead.
*   **GC Impact:** Primarily driven by the number of objects on the heap and the pointer density.
    *   **High Allocation Rate:** Leads to more frequent GC cycles, consuming more CPU.
    *   **Large Live Heap:** Each GC cycle takes longer to mark.
*   **Reducing GC Pressure:**
    *   **Reduce Allocations:** The most effective way. Fewer objects means less work for the GC. Use profiling (heap) to find allocation hotspots.
    *   **Use `sync.Pool`:** Reuse objects (like buffers) to avoid allocating/deallocating them frequently. Objects in the pool *can* be garbage collected. Reset pooled objects before use.
    *   **Avoid unnecessary pointers:** Structs with fewer pointers are scanned faster by the GC.
*   **Forcing GC:** `runtime.GC()` - Primarily for debugging or testing memory usage. Avoid in production code.
*   **Returning Memory to OS:** `debug.FreeOSMemory()` - Tries to return unused memory pages to the OS. Can be useful for long-running services with fluctuating memory usage, but has its own overhead.

### 3. CPU Usage & Scheduling

*   **Goroutines:** Lightweight concurrent functions. Multiplexed onto OS threads (M:N scheduling).
*   **Go Scheduler:** Manages goroutines, assigning them to run on OS threads (`M`).
*   **`GOMAXPROCS` Environment Variable:**
    *   Sets the maximum number of OS threads that can execute user-level Go code simultaneously.
    *   Default: Number of logical CPU cores available.
    *   `runtime.GOMAXPROCS(n)` can set/get this at runtime.
    *   Typically, the default is optimal. Changing it might help if goroutines are blocking excessively on CGO calls or non-yielding system calls, but requires careful testing.
*   **CPU Profiling:** Use `pprof` to identify functions consuming the most CPU time (see Section 4).
*   **Potential CPU Issues:**
    *   **Busy Loops:** Tight loops without I/O or channel operations can hog an OS thread.
    *   **High Contention:** Excessive locking (`sync.Mutex`) can lead to goroutines spinning while waiting.
    *   **GC CPU:** High allocation rates lead to significant CPU usage by the GC.
    *   **Inefficient Algorithms:** Poor algorithmic complexity dominates performance.
*   **Yielding:** `runtime.Gosched()` - Yields the processor, allowing other goroutines to run. Rarely needed as the scheduler handles preemption at function calls, channel ops, etc.

### 4. Performance Analysis Tools

Profiling and tracing are essential *before* optimizing. **Don't guess, measure!**

*   **Profiling (`pprof`):** Collects statistical samples of program execution.
    *   **Types:**
        *   `cpu`: CPU usage (function time).
        *   `heap`: Memory allocations (live objects `inuse_space`/`inuse_objects`, allocated objects `alloc_space`/`alloc_objects`).
        *   `goroutine`: Stack traces of all current goroutines.
        *   `mutex`: Stack traces of goroutines contending for mutexes.
        *   `block`: Stack traces of goroutines blocked on synchronization primitives (channels, mutexes).
    *   **Enabling:**
        *   Web Server: `import _ "net/http/pprof"` (registers handlers under `/debug/pprof/`)
        *   Code: `runtime/pprof` package (write profiles to files).
    *   **Analyzing:** `go tool pprof <binary> <profile_file>` or `go tool pprof http://localhost:port/debug/pprof/profile?seconds=30`
        *   Commands: `top`, `list <func>`, `web` (generates SVG graph, requires graphviz), `peek <func>`.
        *   **Flame Graphs:** Excellent for visualizing CPU usage hierarchy (`go tool pprof -http=:8081 ...`).
*   **Execution Tracer (`go tool trace`):** Captures fine-grained runtime events (GC, scheduler, goroutine execution, syscalls, heap timeline). Excellent for diagnosing latency issues, GC pauses, and scheduler contention.
    *   **Enabling:**
        *   Web Server: Fetch `/debug/pprof/trace?seconds=5`
        *   Code: `runtime/trace` package.
    *   **Analyzing:** `go tool trace <trace_file>` (opens browser UI).
        *   Key views: "View trace", "Goroutine analysis", "Network blocking profile", "Synchronization blocking profile".
*   **Compiler Flags:**
    *   `-gcflags="-m"`: Show escape analysis and inlining decisions.
    *   `-gcflags="-S"`: Output assembly code.

### 5. Benchmarking (`testing` package)

Measure the performance of specific code sections.

*   **Benchmark Functions:**
    ```go
    import "testing"

    func BenchmarkMyFunction(b *testing.B) {
        // Setup code (not measured)
        setupData := createData() 

        b.ResetTimer() // Start timing
        for i := 0; i < b.N; i++ { // Loop b.N times
            MyFunction(setupData) // The code to benchmark
        }
        b.StopTimer() // Optional: Stop timing for cleanup

        // Cleanup code (not measured)
    }
    ```
*   **`b.N`:** The benchmark runner dynamically adjusts `b.N` until the benchmark runs for a measurable duration (default 1 second). Your code *must* loop `b.N` times.
*   **Running Benchmarks:**
    *   `go test -bench=.`: Run all benchmarks in the current package.
    *   `go test -bench=MyFunction`: Run a specific benchmark by regex.
    *   `go test -benchmem`: Show memory allocation stats (allocs/op, B/op). **Crucial!**
    *   `go test -benchtime=3s`: Run benchmarks for a minimum of 3 seconds.
    *   `go test -count=5`: Run each benchmark 5 times.
    *   `go test -cpu=1,2,4`: Run benchmarks with `GOMAXPROCS=1`, `GOMAXPROCS=2`, etc.
*   **Avoiding Compiler Optimizations:** Ensure the code inside the loop isn't optimized away.
    *   Assign results to a package-level variable.
    *   Use `runtime.KeepAlive()` on inputs if necessary.
*   **Comparing Benchmarks:** Use tools like `benchstat` (`go install golang.org/x/perf/cmd/benchstat@latest`) to analyze performance changes between runs.
    ```bash
    go test -bench=. -count=10 > old.txt
    # Make code changes
    go test -bench=. -count=10 > new.txt
    benchstat old.txt new.txt 
    ```

### 6. Common Performance Pitfalls & Optimization Patterns

1.  **Excessive Allocations:** (Use heap profiling!)
    *   **String Concatenation:** Use `strings.Builder` instead of `+` or `fmt.Sprintf` in loops.
    *   **`[]byte` <-> `string` Conversions:** Avoid unnecessary conversions; they allocate.
    *   **Passing by Value:** Large structs passed by value cause copying. Pass by pointer, but be aware of potential escape analysis implications.
    *   **`defer` in Loops:** `defer` allocates. Avoid inside tight loops; refactor if needed.
    *   **Using `interface{}`/`any`:** Can cause heap allocations for underlying values and involve type assertions/reflection overhead. Use generics where appropriate.
2.  **Inefficient Map/Slice Operations:**
    *   Pre-allocate capacity (`make` with size hint).
    *   Ranging over large arrays/slices by value creates copies (less common now, but be aware).
3.  **Lock Contention (`sync.Mutex`/`RWMutex`):** (Use mutex/block profiling!)
    *   Keep critical sections (code between `Lock`/`Unlock`) as short as possible.
    *   Use `sync.RWMutex` if reads are much more frequent than writes.
    *   Consider finer-grained locking or lock-free algorithms (advanced).
4.  **JSON Handling:**
    *   `encoding/json` uses reflection, which has overhead. For high-performance scenarios, consider code-generation based libraries (`ffjson`, `easyjson`, `json-iterator/go`), but adds build complexity.
5.  **I/O:**
    *   Use buffered I/O (`bufio.Reader`, `bufio.Writer`) to reduce syscall overhead for frequent small reads/writes.
6.  **Context Switching:** Extremely high numbers of active, non-blocking goroutines can increase scheduling overhead. Worker pools can help manage concurrency.
7.  **CGO Overhead:** Calls between Go and C have significant overhead compared to native Go calls. Minimize CGO calls in performance-critical paths.

### 7. General Performance Philosophy

*   **Profile First:** Don't optimize prematurely. Use `pprof` and `trace` to identify actual bottlenecks.
*   **Focus on Allocations:** Reducing heap allocations often yields significant performance improvements by reducing GC pressure and allocation overhead.
*   **Understand Algorithms:** A better algorithm (e.g., O(n log n) vs O(n^2)) usually beats micro-optimizations.
*   **Write Clear Code First:** Optimize only when necessary and after profiling confirms a bottleneck. Complex, "optimized" code is harder to maintain.
*   **Benchmark Changes:** Always measure the impact of your optimizations using benchmarks.

---

This cheatsheet provides a starting point. Performance tuning is often an iterative process of profiling, identifying bottlenecks, hypothesizing solutions, implementing, and benchmarking.
