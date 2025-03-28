## Go Testing: Advanced Techniques

This cheatsheet focuses on patterns, tools, and techniques for writing more effective, maintainable, and comprehensive tests in Go.

### 1. Table-Driven Tests

A standard pattern for testing multiple input/output combinations of a function concisely.

*   **Structure:** Define a slice of anonymous structs, each representing a test case. Iterate over the slice, running each case as a subtest.
*   **Subtests (`t.Run`)**: Essential for table-driven tests.
    *   Isolates each test case.
    *   Provides clearer output (e.g., `TestMyFunction/case_name`).
    *   Allows targeted execution (`go test -run TestMyFunction/case_name`).
    *   Works well with `t.Parallel()` for concurrent execution of independent cases.
*   **Example:**
    ```go
    func TestAdd(t *testing.T) {
        testCases := []struct {
            name string // Name for the subtest
            a    int
            b    int
            want int
        }{
            {"zero plus zero", 0, 0, 0},
            {"positive plus positive", 2, 3, 5},
            {"positive plus negative", 5, -2, 3},
            // Add more cases...
        }

        for _, tc := range testCases {
            tc := tc // Capture range variable for parallel tests (Go 1.22+ may not need this)
            t.Run(tc.name, func(t *testing.T) {
                // t.Parallel() // Optional: Run subtests concurrently if they are independent
                got := Add(tc.a, tc.b)
                if got != tc.want {
                    t.Errorf("Add(%d, %d) = %d; want %d", tc.a, tc.b, got, tc.want)
                }
            })
        }
    }
    ```

### 2. Test Setup & Teardown

*   **`TestMain(m *testing.M)`:** (Per-package setup/teardown)
    *   An optional function that, if defined in a test file, runs *once* for the entire package.
    *   Used for expensive setup (e.g., starting a database container, creating global resources) or teardown.
    *   **Must call `m.Run()`** to execute the actual tests. The exit code from `m.Run()` should be passed to `os.Exit()`.
    ```go
    func TestMain(m *testing.M) {
        fmt.Println("Setting up package resources...")
        // Setup code here

        exitCode := m.Run() // Run all tests in the package

        fmt.Println("Tearing down package resources...")
        // Teardown code here

        os.Exit(exitCode)
    }
    ```
*   **`t.Cleanup(f func())`:** (Per-test or per-subtest teardown, Go 1.14+)
    *   Registers a function `f` to be called when the test or subtest completes (passes, fails, or panics).
    *   Functions registered with `t.Cleanup` run in LIFO (Last-In, First-Out) order, similar to `defer`.
    *   **Preferred over manual `defer`** for teardown within tests, as it runs even if `t.Fatal` or `t.Skip` is called.
    ```go
    func TestWithTempFile(t *testing.T) {
        tmpFile, err := os.CreateTemp("", "mytest*.txt")
        if err != nil {
            t.Fatalf("Failed to create temp file: %v", err)
        }
        // Cleanup runs when TestWithTempFile (or its subtest) finishes
        t.Cleanup(func() { 
            os.Remove(tmpFile.Name()) 
            tmpFile.Close()
            fmt.Println("Cleaned up temp file:", tmpFile.Name())
        })

        // ... test logic using tmpFile ...
        tmpFile.WriteString("hello")
    }
    ```

### 3. Mocking & Interfaces

Essential for isolating the code under test from external dependencies (databases, APIs, filesystem, time).

*   **Design with Interfaces:** The most crucial step. Define interfaces for dependencies rather than using concrete types directly.
    ```go
    // Define an interface for dependency
    type UserNotifier interface {
        Notify(userID int, message string) error
    }

    // Your service depends on the interface, not a concrete type
    type UserService struct {
        notifier UserNotifier
        // ... other fields ...
    }

    func (s *UserService) RegisterUser(id int) {
        // ... register user logic ...
        err := s.notifier.Notify(id, "Welcome!")
        if err != nil { /* handle error */ }
    }
    ```
*   **Manual Mock:** Create a simple struct that implements the interface for test purposes.
    ```go
    type mockNotifier struct {
        NotifyCalled bool
        LastUserID   int
        LastMessage  string
        ReturnError  error
    }

    func (m *mockNotifier) Notify(userID int, message string) error {
        m.NotifyCalled = true
        m.LastUserID = userID
        m.LastMessage = message
        return m.ReturnError
    }

    func TestUserService_RegisterUser(t *testing.T) {
        mock := &mockNotifier{}
        service := &UserService{notifier: mock}

        service.RegisterUser(123)

        if !mock.NotifyCalled { t.Error("notifier.Notify was not called") }
        if mock.LastUserID != 123 { t.Errorf("...") }
        if mock.LastMessage != "Welcome!" { t.Errorf("...") }
    }
    ```
*   **Mocking Libraries:** Automate mock creation and expectation setting.
    *   **`testify/mock`:** Popular choice. Define expectations at runtime.
        *   Requires `go get github.com/stretchr/testify/mock`.
        *   Embed `mock.Mock` in your mock struct. Set expectations using `On(...)`, `Return(...)`. Assert expectations with `AssertExpectations(t)`.
        ```go
        import "github.com/stretchr/testify/mock"
        
        type MockNotifier struct {
            mock.Mock 
        }

        func (m *MockNotifier) Notify(userID int, message string) error {
            args := m.Called(userID, message)
            return args.Error(0) 
        }

        func TestUserServiceWithTestify(t *testing.T) {
            mockNotifier := new(MockNotifier)
            service := &UserService{notifier: mockNotifier}

            // Expect Notify to be called once with 123 and "Welcome!"
            // and return nil error
            mockNotifier.On("Notify", 123, "Welcome!").Return(nil).Once()

            service.RegisterUser(123)

            // Verify that expectations were met
            mockNotifier.AssertExpectations(t) 
        }
        ```
    *   **`gomock`:** (From Google) Generates mock code from interfaces using `mockgen`. More type-safe, strict expectation matching. Often used with `controller`.

### 4. Testing HTTP Handlers & Clients

*   **`net/http/httptest` Package:**
    *   **`httptest.NewRecorder()`:** Implements `http.ResponseWriter` to record the status code, headers, and body written by a handler.
    *   **`httptest.NewRequest()` / `http.NewRequest()`:** Create `*http.Request` objects for testing handlers. Use `http.NewRequest` for more control (context, body).
        ```go
        func TestMyHandler(t *testing.T) {
            req := httptest.NewRequest("GET", "/path?param=value", nil)
            // Add headers if needed: req.Header.Set(...)
            rr := httptest.NewRecorder()

            myHandler := http.HandlerFunc(MyActualHandler)
            myHandler.ServeHTTP(rr, req) // Call the handler

            // Check status code, headers, body
            if rr.Code != http.StatusOK { t.Errorf("...") }
            if body := rr.Body.String(); body != "expected" { t.Errorf("...") }
        }
        ```
    *   **`httptest.NewServer(handler)`:** Starts a real HTTP server on a system-chosen port using the provided handler. Used for testing **HTTP clients**. Returns `*httptest.Server`.
        ```go
        func TestMyAPIClient(t *testing.T) {
            // Mock server responding like the real API
            mockAPIServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                if r.URL.Path == "/users/1" {
                    w.WriteHeader(http.StatusOK)
                    w.Write([]byte(`{"id": 1, "name": "Alice"}`))
                    return
                }
                w.WriteHeader(http.StatusNotFound)
            }))
            defer mockAPIServer.Close() // Ensure server is shut down

            // Create client pointing to the mock server
            myClient := NewMyAPIClient(mockAPIServer.URL) // Your client needs to accept base URL

            user, err := myClient.GetUser(1)
            // Assertions on user and err...
        }
        ```

### 5. Filesystem Abstraction

Avoid interacting directly with the real filesystem in unit tests.

*   **`testing/fstest` (Go 1.16+):** Provides tools for creating in-memory filesystems, primarily read-only.
    *   `fstest.MapFS`: A map-based filesystem (`map[string]*fstest.MapFile`).
    ```go
    import "testing/fstest"

    func TestProcessConfig(t *testing.T) {
        // Create an in-memory filesystem
        mapFS := fstest.MapFS{
            "config/settings.json": {Data: []byte(`{"port": 8080}`)},
            "data/file.txt":        {Data: []byte("content")},
        }

        // Pass the fs.FS interface to your function (it needs to accept fs.FS)
        config, err := ReadConfigFromFS(mapFS, "config/settings.json")
        // Assertions...
    }

    // Your function needs to accept fs.FS (or io/fs.FS)
    func ReadConfigFromFS(fsys fs.FS, path string) (*Config, error) {
        data, err := fs.ReadFile(fsys, path)
        // ... parse data ...
    }
    ```
*   **`afero` library:** (Third-party) A more comprehensive filesystem abstraction library supporting read/write operations and various backend implementations (memory, OS, etc.).

### 6. Fuzz Testing (Go 1.18+)

Automatically generates inputs to find edge cases and potential bugs/panics that regular unit tests might miss.

*   **Signature:** `func FuzzXxx(f *testing.F)`
*   **Seed Corpus (`f.Add(...)`):** Provide initial interesting inputs.
*   **Fuzz Target (`f.Fuzz(...)`):** A function accepting `*testing.T` and generated input arguments (types matching the seed corpus).
*   **Running:** `go test -fuzz=FuzzXxx [-fuzztime=30s]`
```go
func FuzzReverse(f *testing.F) {
    // Seed corpus: Provide initial examples
    f.Add("hello")
    f.Add("世界")
    f.Add("")

    // Fuzz target: Receives generated strings
    f.Fuzz(func(t *testing.T, original string) {
        reversed := Reverse(original) // Your function under test
        doubleReversed := Reverse(reversed)
        if original != doubleReversed {
            t.Errorf("Reverse(Reverse(%q)) != %q", original, original)
        }
        // Add other invariants/checks...
        if utf8.ValidString(original) && !utf8.ValidString(reversed) {
             t.Errorf("Reverse produced invalid UTF-8 string: %q", reversed)
        }
    })
}
```

### 7. Parallel Tests (`t.Parallel()`)

Run independent tests or subtests concurrently to speed up the test suite.

*   Call `t.Parallel()` at the beginning of the test function or subtest (`t.Run`).
*   **Warning:** Tests running in parallel **must not** share or modify the same state without proper synchronization (mutexes, channels), otherwise data races will occur. Run tests with the `-race` flag!
*   Parent tests only complete *after* all their parallel subtests have finished.

### 8. Test Coverage

Measure which parts of your code are exercised by tests.

*   `go test -cover`: Show basic coverage percentage per package.
*   `go test -coverprofile=coverage.out`: Generate a coverage profile file.
*   `go tool cover -html=coverage.out`: Open an HTML report in the browser showing line-by-line coverage. Useful for identifying untested code paths.

### 9. Helper Functions (`t.Helper()`)

Reduce boilerplate and improve readability in tests.

*   Call `t.Helper()` at the beginning of your test helper function.
*   This tells the testing framework that this function is a test helper. When `t.Error/Fatal` is called *inside* the helper, the reported line number will be the line where the helper was called in the main test, not the line inside the helper itself.
```go
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper() // Mark this as a helper
    if got != want {
        t.Errorf("Assertion failed: got %v, want %v", got, want)
    }
}

func TestSomething(t *testing.T) {
    result := DoSomething()
    assertEqual(t, result, "expected_value") // Error reported on this line
}
```

### 10. Example Functions (`example_test.go`)

Serve as documentation, usage examples, and runnable tests.

*   Function name starts with `Example`.
*   Use `// Output:` comment at the end to specify expected standard output. The test runner compares actual stdout with this comment.
```go
func ExampleAdd() {
    sum := Add(2, 3)
    fmt.Println(sum)
    // Output: 5 
}
```

---

Applying these techniques helps create more reliable, maintainable, and faster Go test suites. Remember to design for testability from the start, primarily by using interfaces for dependencies.
