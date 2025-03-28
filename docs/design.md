## Go Idiomatic Principles & Design

This cheatsheet covers conventions, best practices, and design philosophies for writing idiomatic Go code, focusing on clarity, simplicity, and maintainability.

### 1. Core Philosophy

*   **Simplicity:** Prefer simple, explicit code over complex or "clever" solutions. Readability counts. "Less is exponentially more."
*   **Readability:** Code is read far more often than it's written. Optimize for clarity. Use `gofmt` religiously.
*   **Explicitness:** Avoid magic. Make control flow, dependencies, and side effects obvious.
*   **Composition over Inheritance:** Use embedding and interfaces instead of traditional class inheritance.
*   **Concurrency:** Embrace Go's built-in concurrency features (goroutines, channels), but use them judiciously. "Share memory by communicating."

### 2. Naming Conventions

*   **Packages:**
    *   Short, concise, lowercase names. Usually single words.
    *   Should be descriptive of the package's purpose (e.g., `http`, `json`, `time`).
    *   Avoid generic names like `util`, `common`, `helpers`, `base`. Group related functionality by domain.
    *   The name users see when importing should match the directory name.
*   **Variables:**
    *   Use `mixedCaps` or `camelCase`.
    *   Short names (e.g., `i`, `k`, `v`, `buf`) are idiomatic within small scopes.
    *   Longer, more descriptive names are better for wider scopes or complex logic.
*   **Functions & Methods:**
    *   Use `mixedCaps`. Exported names start with an uppercase letter.
    *   Names should indicate what they do. If it mutates state, often a verb; if it returns info, often a noun or "GetX".
*   **Interfaces:**
    *   Often named with an "-er" suffix for single-method interfaces (e.g., `io.Reader`, `fmt.Stringer`).
    *   For multi-method interfaces, choose a noun that represents the concept (e.g., `http.Handler`, `sql.DB`).
*   **Constants:**
    *   Use `mixedCaps`, just like variables. There's no convention for `ALL_CAPS` in Go (unlike C/Java).
*   **Acronyms:** Keep acronyms consistent (e.g., `ServeHTTP`, `userID`, `parseURL`, not `ServeHttp`, `userId`, `parseUrl`). `ID` is preferred over `Id`.

### 3. Package Design

*   **Cohesion:** Group related types and functions together within a package. A package should serve a clear, single purpose.
*   **Minimalism:** Export only what is necessary for users of the package. Keep the public API surface small.
*   **`internal` Directory:** Code inside `internal/` can only be imported by code within the same module rooted one level above `internal/` (e.g., `/myproject/internal/auth` can be imported by `/myproject/server` but not by `/otherproject`). Use this to hide implementation details you don't want to expose publicly *within your own module*.
*   **Avoid Cyclic Dependencies:** Package A cannot import Package B if Package B imports Package A (directly or indirectly). Structure packages to form a Directed Acyclic Graph (DAG). Use interfaces to break cycles if needed.
*   **Package Comments:** Write a good package comment (`// Package mypkg does...`) explaining the package's role. `godoc` uses this.

### 4. Function & Method Design

*   **Small Functions:** Keep functions focused on a single task.
*   **Parameter Passing:**
    *   Pass small, immutable values or basic types by value.
    *   Pass large structs by pointer to avoid copying costs.
    *   Pass maps, slices, channels by value (they are reference types internally, like pointers to underlying data).
    *   If a method needs to modify the receiver, use a pointer receiver (`func (s *MyStruct) Modify()`).
    *   Be consistent with receiver types (pointer or value) for a given type.
*   **Return Values:**
    *   Use explicit `error` return values for expected errors (see Error Handling).
    *   Multiple return values are common and idiomatic.
    *   Named return values can improve clarity in short functions or when documenting return values, but use sparingly as they can sometimes reduce readability in longer functions.
*   **Defer:** Use `defer` for cleanup actions (closing files, unlocking mutexes, etc.) close to where the resource is acquired.

### 5. API Design

*   **Accept Interfaces, Return Structs:** A common guideline (not a strict rule). Functions often accept interfaces to allow flexibility for callers (dependency inversion), and return concrete types (structs) as they typically know exactly what they are creating.
*   **Zero Value:** Strive to make the zero value of your types useful (e.g., `sync.Mutex`, `bytes.Buffer`). Avoid requiring explicit initialization if the zero state is meaningful.
*   **Constructors:** Use `New...` functions (e.g., `NewClient`, `NewReader`) to create instances of types, especially if initialization is complex or requires validation.
*   **Consistency:** Maintain consistency in naming, parameter order, and behavior across your API.
*   **Context:** For functions involving blocking operations, I/O, or external requests, accept `context.Context` as the first argument (`func DoSomething(ctx context.Context, ...)`).
*   **Concurrency Safety:** Clearly document whether your types and functions are safe for concurrent use. If not, users must provide external synchronization.

### 6. Error Handling

*   **Explicit Checks:** Always check returned errors (`if err != nil { ... }`).
*   **`error` Interface:** Return errors using the standard `error` interface.
*   **Error Wrapping:** Use `fmt.Errorf("operation failed: %w", err)` with the `%w` verb to wrap underlying errors, preserving context.
*   **Checking Errors:**
    *   Use `errors.Is(err, targetErr)` to check if an error *is* (or wraps) a specific sentinel error (e.g., `io.EOF`).
    *   Use `errors.As(err, &targetType)` to check if an error *is* (or wraps) a specific custom error type, and retrieve it.
*   **Sentinel Errors:** Predefined exported error variables (e.g., `var ErrNotFound = errors.New("not found")`). Use sparingly, as they create dependencies. Custom error types or checking error messages/codes can be alternatives.
*   **Panic vs. Error:** Use `panic` only for truly exceptional, unrecoverable situations (e.g., programming errors like nil pointer dereference where it shouldn't be possible, impossible state). Regular, expected errors (network issues, file not found, bad input) should *always* be handled via `error` return values.

### 7. Interfaces

*   **Small Interfaces:** Prefer small, focused interfaces (like `io.Reader`, `io.Writer`). The smaller the interface, the more types can satisfy it, leading to greater flexibility.
*   **Implicit Satisfaction:** Types satisfy interfaces implicitly by implementing the required methods. No `implements` keyword.
*   **Purpose:** Use interfaces to define behavior, abstract away concrete implementations, and enable dependency injection/testing.

### 8. Concurrency

*   **Channels for Communication:** Prefer using channels to communicate data between goroutines ("share memory by communicating").
*   **`sync` for State Protection:** Use `sync` package primitives (`Mutex`, `RWMutex`) primarily to protect shared state *within* a component accessed by multiple goroutines.
*   **`context.Context` for Cancellation:** Use `context` to handle cancellation, timeouts, and deadlines across goroutines and API boundaries.
*   **Keep it Simple:** Avoid overly complex concurrent designs unless necessary. Simple, well-understood patterns are easier to reason about.

### 9. Testing

*   **Table-Driven Tests:** Use tables (slices of test cases) for testing multiple inputs/outputs for the same function.
*   **Package per Directory:** Place test files (`_test.go`) in the same package/directory as the code they test.
*   **Subtests:** Use `t.Run` to create hierarchical subtests for better organization and reporting.
*   **`testing/fstest`, `net/http/httptest`:** Leverage standard library testing helpers.
*   **Mocking:** Use interfaces to allow mocking dependencies during tests.

### 10. Documentation

*   **`godoc` Format:** Write comments using the standard `godoc` format.
    *   Start sentences with the name of the thing being documented (e.g., `// MyFunc performs...`).
    *   Use package comments (`// Package http provides...`).
    *   Document all exported identifiers (types, functions, variables, constants).
*   **Examples:** Provide `Example` functions (`example_test.go`) to show usage. These are run as tests and appear in the documentation.

### 11. Tooling

*   **`gofmt`:** Format your code automatically. Non-negotiable. Integrate into your editor and CI.
*   **`go vet`:** Run `go vet ./...` to catch common mistakes and suspicious constructs.
*   **Linters (`golangci-lint`):** Use additional linters to enforce style and find potential issues beyond `go vet`.

---

Adhering to these principles will make your Go code more effective, maintainable, and easier for other Go developers to understand and contribute to.
