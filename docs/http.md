## Go HTTP (Server, Client, Routing)

This cheatsheet covers building HTTP servers, making HTTP requests, and related patterns in Go using primarily the standard `net/http` package.

### 1. HTTP Server Basics (`net/http`)

*   **Starting a Simple Server:**
    ```go
    package main

    import (
        "fmt"
        "log"
        "net/http"
    )

    func helloHandler(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    }

    func main() {
        // Register handler function for the "/" path
        http.HandleFunc("/", helloHandler) 

        // Start the server on port 8080
        // ListenAndServe blocks, so errors are typically fatal
        log.Println("Starting server on :8080")
        err := http.ListenAndServe(":8080", nil) // nil means use DefaultServeMux
        if err != nil {
            log.Fatal("ListenAndServe: ", err)
        }
    }
    ```
*   **Handlers:** Types that implement the `http.Handler` interface.
    ```go
    type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
    }
    ```
*   **Handler Functions:** Use `http.HandlerFunc` adapter to use plain functions as handlers.
    ```go
    func myHandler(w http.ResponseWriter, r *http.Request) { /* ... */ }
    http.Handle("/path", http.HandlerFunc(myHandler)) 
    // http.HandleFunc is syntactic sugar for the above
    http.HandleFunc("/path", myHandler) 
    ```
*   **`http.ResponseWriter`:** Interface used by a handler to construct the HTTP response.
    *   `w.WriteHeader(statusCode int)`: Sets the HTTP status code (e.g., `http.StatusOK`, `http.StatusBadRequest`). Call *before* `w.Write`. If not called, defaults to `http.StatusOK`.
    *   `w.Header().Set(key, value string)`: Sets response headers. Call *before* `w.WriteHeader` or `w.Write`.
    *   `w.Write(body []byte)`: Writes the response body. Implicitly calls `WriteHeader(http.StatusOK)` if not already called.
*   **`http.Request`:** Struct representing the incoming HTTP request.
    *   `r.Method`: Request method (e.g., "GET", "POST").
    *   `r.URL`: Parsed URL (`*url.URL`).
        *   `r.URL.Path`: Request path (e.g., "/users/123").
        *   `r.URL.Query()`: Parsed query parameters (`url.Values`, a `map[string][]string`).
    *   `r.Header`: Request headers (`http.Header`, a `map[string][]string`). Use `r.Header.Get(key)` for single value lookup.
    *   `r.Body`: Request body (`io.ReadCloser`). **Important:** Must be closed by the handler (`defer r.Body.Close()`).
    *   `r.Context()`: Request context (`context.Context`), useful for cancellation, deadlines, and passing request-scoped values.

### 2. Handling Requests: Reading Data

*   **Reading Query Parameters:**
    ```go
    // For URL like /search?q=golang&limit=10
    query := r.URL.Query() 
    searchTerm := query.Get("q") // Gets the first value for "q"
    limitStr := query.Get("limit") // Returns "" if not present
    allQ := query["q"] // Gets slice of all values for "q"
    
    // Or use FormValue (parses query AND form data if needed)
    searchTerm = r.FormValue("q") 
    ```
*   **Reading Path Parameters (Requires Router):** Standard `http.ServeMux` doesn't support named path parameters like `/users/{id}` directly. Use a third-party router (see Section 3).
*   **Reading Headers:**
    ```go
    authHeader := r.Header.Get("Authorization")
    contentType := r.Header.Get("Content-Type")
    ```
*   **Reading Request Body:**
    ```go
    // Always close the body
    defer r.Body.Close() 
    
    // Read the entire body (use with caution for large bodies)
    bodyBytes, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Failed to read body", http.StatusInternalServerError)
        return
    }
    bodyString := string(bodyBytes)
    
    // Decode JSON body
    var target struct { // Define your target struct
        Name string `json:"name"`
        Age  int    `json:"age"`
    }
    err = json.NewDecoder(r.Body).Decode(&target)
    if err != nil {
        // Handle JSON decoding errors (e.g., bad format, EOF)
        http.Error(w, "Invalid JSON format", http.StatusBadRequest)
        return
    }
    fmt.Printf("Decoded: %+v\n", target)
    ```

### 3. Handling Requests: Writing Responses

*   **Setting Status Code & Headers:**
    ```go
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom-Header", "some-value")
    w.WriteHeader(http.StatusCreated) // Set status to 201 Created
    ```
*   **Writing Plain Text:**
    ```go
    fmt.Fprintf(w, "User %s created successfully", userName)
    // or w.Write([]byte("User created"))
    ```
*   **Writing JSON:**
    ```go
    w.Header().Set("Content-Type", "application/json")
    response := map[string]interface{}{
        "message": "Success",
        "id":      123,
    }
    err := json.NewEncoder(w).Encode(response)
    if err != nil {
        // Log error, but response might be partially written
        log.Printf("Error encoding JSON response: %v", err) 
        // Avoid writing http.Error here if headers/body already sent
    }
    ```
*   **Sending Errors:**
    ```go
    // Simple error response
    http.Error(w, "Resource not found", http.StatusNotFound) 
    
    // JSON error response
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(w).Encode(map[string]string{"error": "Invalid input provided"})
    ```

### 4. Routing

*   **`http.DefaultServeMux` (Standard Library):**
    *   Used when `nil` is passed to `http.ListenAndServe`.
    *   Matches based on longest path prefix.
    *   `/path/` matches subtree, `/path` matches exact path only.
    *   From Go 1.22: 
        * Method matching: mux.HandleFunc("GET /path", getHandler).
        *  Wildcards: mux.HandleFunc("/users/{id}", userHandler).
        * Precedence rules (more specific patterns win).
    ```go
            mux := http.NewServeMux() // Create a new ServeMux

            // Method-specific route
            mux.HandleFunc("GET /items", func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprintln(w, "Fetching all items")
            })
            mux.HandleFunc("POST /items", func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprintln(w, "Creating an item")
            })

            // Route with path parameter/wildcard
            mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
                itemID := r.PathValue("id") // Access path parameter (Go 1.22+)
                fmt.Fprintf(w, "Fetching item with ID: %s\n", itemID)
            })

            // Route with trailing slash and wildcard
            mux.HandleFunc("GET /files/{$}", func(w http.ResponseWriter, r *http.Request) {
                // Trailing slash indicates subtree match. '{$}' captures the rest.
                filePath := r.PathValue("$") 
                fmt.Fprintf(w, "Accessing file path: %s\n", filePath)
            })

            // Basic root handler
            mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) { /* ... */ })

            log.Println("Starting server with enhanced Mux on :8080")
            // Use the new mux instead of nil (DefaultServeMux)
            err := http.ListenAndServe(":8080", mux) 
            if err != nil {
                log.Fatal(err)
            }
    ```
*   **Third-Party Routers (Muxes):** Provide more features like path parameters, method matching, middleware integration. Popular choices:
    *   `gorilla/mux`: Feature-rich and widely used.
    *   `chi`: Lightweight, composable, good middleware support.
    *   `gin-gonic/gin`: Fast, includes middleware framework, popular but more opinionated.
    *   **Example (`chi`):**
        ```go
        import "github.com/go-chi/chi/v5"

        func main() {
            r := chi.NewRouter()
            // Add middleware (e.g., logging)
            r.Use(middleware.Logger) 

            r.Get("/", func(w http.ResponseWriter, r *http.Request) { /* ... */ })
            r.Post("/users", createUserHandler)
            r.Get("/users/{userID}", getUserHandler) // Path parameter

            log.Println("Starting server with Chi on :8080")
            http.ListenAndServe(":8080", r)
        }

        func getUserHandler(w http.ResponseWriter, r *http.Request) {
            userID := chi.URLParam(r, "userID") // Extract path parameter
            fmt.Fprintf(w, "Fetching user %s", userID)
        }
        ```

### 5. Middleware (HTTP Decorators)

A pattern for wrapping handlers to add cross-cutting concerns (logging, auth, compression, CORS, etc.).

*   **Common Signature:** `func(http.Handler) http.Handler`
*   **Example: Simple Logging Middleware**
    ```go
    func loggingMiddleware(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            log.Printf("Started %s %s", r.Method, r.URL.Path)
            
            // Call the next handler in the chain
            next.ServeHTTP(w, r) 
            
            log.Printf("Completed %s in %v", r.URL.Path, time.Since(start))
        })
    }

    func main() {
        mux := http.NewServeMux() // Use a non-default mux
        mux.HandleFunc("/", helloHandler)

        // Wrap the entire mux with the middleware
        loggedMux := loggingMiddleware(mux) 

        log.Println("Starting server with logging on :8080")
        http.ListenAndServe(":8080", loggedMux)
    }
    ```
*   Many third-party routers have built-in support for easier middleware chaining.
* **Note on logging:** the `log/slog` package (Go 1.21) is the new standard library recommendation for structured logging, which is ideal for HTTP request logging middleware. Brief example of `slog` usage:
    ```go
    // Inside a logging middleware:
    slog.Info("request received", 
        "method", r.Method, 
        "path", r.URL.Path,
        "remote_addr", r.RemoteAddr,
        "user_agent", r.UserAgent(),
    )
    // ... call next handler ...
    slog.Info("request completed", "status", recordedStatusCode, "duration", time.Since(start))
    ```

### 6. HTTP Client (`net/http`)

*   **Default Client:** Convenient for simple requests. **Warning:** Has *no timeouts* by default.
    ```go
    resp, err := http.Get("https://example.com/")
    if err != nil { /* handle error */ }
    defer resp.Body.Close() // IMPORTANT: Close body even if not reading

    if resp.StatusCode != http.StatusOK { /* handle non-200 status */ }

    bodyBytes, err := io.ReadAll(resp.Body)
    // ... process bodyBytes ...
    ```
*   **Custom Client (Recommended):** Create a client for setting timeouts, custom transport, etc.
    ```go
    client := &http.Client{
        Timeout: 10 * time.Second, // Overall request timeout
        // Transport: &http.Transport{ // Customize transport details if needed
        // 	DialContext: (&net.Dialer{
        // 		Timeout:   5 * time.Second, // Connection timeout
        // 		KeepAlive: 30 * time.Second,
        // 	}).DialContext,
        // 	TLSHandshakeTimeout:   10 * time.Second,
        //  MaxIdleConns:          100,
        //  IdleConnTimeout:       90 * time.Second,
        //  ExpectContinueTimeout: 1 * time.Second,
        // },
    }

    resp, err := client.Get("https://example.com/")
    // ... handle resp and err as above ...
    ```
*   **Making POST/Other Requests:**
    ```go
    // POST with JSON body
    jsonData := []byte(`{"name":"Alice", "age":30}`)
    req, err := http.NewRequest("POST", "https://example.com/users", bytes.NewBuffer(jsonData))
    if err != nil { /* handle error */ }
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer your_token")

    // Use context for cancellation/deadlines
    // ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    // defer cancel()
    // req = req.WithContext(ctx)

    resp, err := client.Do(req) // Use custom client
    if err != nil { /* handle error */ }
    defer resp.Body.Close()
    // ... process response ...

    // POST Form data
    formData := url.Values{}
    formData.Set("key1", "value1")
    formData.Set("key2", "value2")
    resp, err = client.PostForm("https://example.com/submit", formData)
    // ... handle resp and err ...
    ```

### 7. HTTPS / TLS

*   **Server:**
    ```go
    // Generate self-signed certs for local dev (don't use in prod):
    // openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

    err := http.ListenAndServeTLS(":8443", "cert.pem", "key.pem", nil) // nil for DefaultServeMux
    if err != nil {
        log.Fatal("ListenAndServeTLS: ", err)
    }
    ```
*   **Production:** Use certificates from a trusted CA (e.g., Let's Encrypt via `golang.org/x/crypto/acme/autocert`).
*   **Client:** By default, the client verifies server certificates against the system's trust store. To skip verification (INSECURE, for testing only):
    ```go
    tr := &http.Transport{
        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
    }
    client := &http.Client{Transport: tr, Timeout: 10 * time.Second}
    // ... use client ...
    ```

### 8. Testing Handlers (`net/http/httptest`)

*   Use `httptest.NewRecorder` to capture the response.
*   Use `httptest.NewRequest` or `http.NewRequest` to create a request.
    ```go
    func TestHelloHandler(t *testing.T) {
        req, err := http.NewRequest("GET", "/", nil)
        if err != nil {
            t.Fatal(err)
        }

        rr := httptest.NewRecorder() // ResponseRecorder implements ResponseWriter
        handler := http.HandlerFunc(helloHandler)

        handler.ServeHTTP(rr, req) // Call the handler directly

        // Check status code
        if status := rr.Code; status != http.StatusOK {
            t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
        }

        // Check response body
        expected := `Hello, World!`
        if rr.Body.String() != expected {
            t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
        }
    }
    ```

### 9. Key Considerations & Best Practices

*   **Always `defer r.Body.Close()` in handlers.**
*   **Always `defer resp.Body.Close()` when using `http.Client`.** Close even if not reading the body to allow connection reuse.
*   **Use a custom `http.Client` with appropriate timeouts.** Avoid the default client in production code.
*   **Handle errors explicitly** (I/O errors, JSON decoding/encoding errors, non-2xx status codes).
*   **Use `context.Context`** from `r.Context()` for cancellation propagation, deadlines, and passing request-scoped data. Pass it to downstream client calls using `http.NewRequestWithContext`.
*   **Consider structured logging** for better observability.
*   **Implement graceful shutdown** to allow ongoing requests to finish before the server exits.
*   **Choose a router** based on your needs for parameters, method matching, and middleware.
*   **Sanitize and validate** all input (query params, path params, headers, body).
*   **Manage server configuration** (ports, timeouts, TLS settings) properly.

---

This provides a comprehensive overview of standard Go HTTP practices. Remember to consult the official `net/http` package documentation for more details.
