## Go Build & Deployment

This cheatsheet covers compiling, packaging (especially with Docker), configuring, and running Go applications in production environments.

### 1. Building the Executable

*   **Basic Build:** Creates an executable in the current directory (if `main` package).
    ```bash
    go build 
    ```
*   **Specify Output Name:**
    ```bash
    go build -o myapp . 
    # or go build -o bin/myapp ./cmd/myapp
    ```
*   **Cross-Compilation:** Building for a different OS/Architecture (essential for deployment).
    ```bash
    # Build for 64-bit Linux (common deployment target)
    GOOS=linux GOARCH=amd64 go build -o myapp-linux-amd64 . 

    # Build for 64-bit ARM Linux (e.g., AWS Graviton)
    GOOS=linux GOARCH=arm64 go build -o myapp-linux-arm64 .

    # Build for Windows
    GOOS=windows GOARCH=amd64 go build -o myapp.exe .
    ```
    *   `GOOS`: Target Operating System (`linux`, `windows`, `darwin`, etc.)
    *   `GOARCH`: Target Architecture (`amd64`, `arm64`, `386`, etc.)
*   **Static Linking:** Go produces statically linked binaries by default on Linux when *not* using CGO. This means the binary includes all necessary Go runtime and library code, making it highly portable and ideal for containers (no need for base OS libraries).
    *   If using CGO (`import "C"`), linking becomes dynamic by default, requiring standard C libraries (like `libc`) on the target system. Use `CGO_ENABLED=0` to disable CGO and force static linking (if your code permits).
        ```bash
        # Disable CGO for potentially smaller/static binary (if no C deps)
        CGO_ENABLED=0 GOOS=linux go build -o myapp-static .
        ```
*   **Embedding Version Information (`ldflags -X`):** Inject build-time variables (like version or commit hash) into the binary.
    ```go
    // In your main package (e.g., main.go or version.go)
    package main
    var (
        Version = "dev" // Default value
        Commit  = "none"
        Date    = "unknown"
    )

    func main() {
        fmt.Printf("Version: %s, Commit: %s, Built: %s\n", Version, Commit, Date)
        // ... rest of your app ...
    }
    ```
    ```bash
    # Get git info
    GIT_COMMIT=$(git rev-parse --short HEAD)
    BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
    APP_VERSION="1.2.3" 

    # Build command using -ldflags -X 'package_path.variable=value'
    go build -ldflags="-X 'main.Version=${APP_VERSION}' -X 'main.Commit=${GIT_COMMIT}' -X 'main.Date=${BUILD_DATE}'" -o myapp .
    ```
    *   Note: `main.Version` is the fully qualified path to the variable.

### 2. Packaging with Docker (Common Practice)

Docker provides isolation and reproducible environments. Multi-stage builds are key for small, secure Go images.

*   **Multi-Stage `Dockerfile`:**
    ```dockerfile
    # ---- Build Stage ----
    # Use a Go image that includes the build toolchain
    FROM golang:1.21-alpine AS builder 

    # Set working directory inside the container
    WORKDIR /app

    # Copy go.mod and go.sum first to leverage Docker layer caching
    COPY go.mod go.sum ./
    RUN go mod download # Download dependencies

    # Copy the rest of the application source code
    COPY . .

    # Build the application as a static binary (disable CGO if possible)
    # Inject version info using ldflags
    ARG APP_VERSION="dev"
    RUN GIT_COMMIT=$(git rev-parse --short HEAD || echo "unknown") && \
        BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') && \
        CGO_ENABLED=0 go build \
        -ldflags="-w -s -X 'main.Version=${APP_VERSION}' -X 'main.Commit=${GIT_COMMIT}' -X 'main.Date=${BUILD_DATE}'" \
        -o /app/myapp .

    # ---- Final Stage ----
    # Use a minimal base image like scratch, distroless, or alpine
    FROM alpine:latest 
    # FROM gcr.io/distroless/static-debian11 AS final # Good alternative for static bins
    # FROM scratch AS final # Absolute minimal, only contains your app + required files

    WORKDIR /app

    # Copy necessary CA certificates (needed for HTTPS requests from the app)
    # This line is needed for Alpine/Scratch if your app makes HTTPS calls
    COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ 

    # Copy only the built application binary from the builder stage
    COPY --from=builder /app/myapp /app/myapp

    # (Optional) Add non-root user for security
    RUN addgroup -S appgroup && adduser -S appuser -G appgroup
    USER appuser 

    # Expose port (if it's a server) - documentation purpose mostly
    EXPOSE 8080 

    # Command to run the application
    ENTRYPOINT ["/app/myapp"] 
    # CMD ["--config", "/etc/myapp/config.yaml"] # Optional default arguments
    ```
*   **Base Image Choices:**
    *   `scratch`: Empty image. Smallest possible size. Only your binary and explicitly copied files (like CA certs). No shell, no tools. Best for truly static binaries.
    *   `distroless` (e.g., `gcr.io/distroless/static-debian11`): From Google. Contains only essential libraries (like libc, CA certs, timezone info) and your app. No shell or package manager. Good balance of size and security.
    *   `alpine`: Small Linux distribution. Includes a shell and package manager (`apk`). Useful if you need OS tools or dynamic linking (e.g., with CGO), but adds overhead and potential vulnerabilities compared to `scratch`/`distroless`. Ensure you install `ca-certificates` if needed.
*   **Build the Image:**
    ```bash
    docker build --build-arg APP_VERSION="1.2.3" -t myapp-image:1.2.3 . 
    ```
*   **Run the Container:**
    ```bash
    docker run -p 8080:8080 --rm myapp-image:1.2.3
    ```

### 3. Reducing Binary Size

Smaller binaries mean faster container image pulls/pushes and deployments.

*   **Strip Debug Information:** Use `ldflags` `-s` (symbol table) and `-w` (DWARF debug info).
    ```bash
    go build -ldflags="-s -w" -o myapp .
    ```
    *   Note: `-s -w` makes debugging harder and prevents some profiling. Apply typically only for the final release build.
*   **`upx` (Optional):** An executable packer. Can significantly reduce size but may increase load time slightly and sometimes triggers antivirus/security scanners. Use with caution.
    ```bash
    # After building with -s -w
    upx --best --lzma ./myapp
    ```

### 4. Embedding Static Assets (`embed` Package - Go 1.16+)

Include static files (HTML templates, JS, CSS, config templates) directly within the Go binary for single-file deployment.

*   **Import:** `import _ "embed"` (for side-effects) or `import "embed"` (to use `embed.FS`).
*   **Directives:** Place `//go:embed` comments above variables.
    ```go
    import (
        "embed"
        "fmt"
        "net/http"
        "io/fs"
    )

    //go:embed static/index.html
    var indexHTML []byte // Embed single file content as bytes

    //go:embed static/styles.css
    var stylesCSS string // Embed single file content as string

    //go:embed static/* templates/*.tmpl
    var staticFiles embed.FS // Embed multiple files/directories into a virtual filesystem

    func main() {
        fmt.Println(string(indexHTML)) 
        fmt.Println(stylesCSS)

        // Serve embedded files over HTTP
        // Create a sub-filesystem rooted at the 'static' directory within embed.FS
        staticFS, _ := fs.Sub(staticFiles, "static")
        http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(staticFS))))

        // Access specific embedded file
        tmplBytes, _ := staticFiles.ReadFile("templates/user.tmpl")
        fmt.Println("Template:", string(tmplBytes))

        http.ListenAndServe(":8080", nil)
    }
    ```

### 5. Configuration Management

How your application gets settings (ports, database URLs, API keys, etc.). Separate config from code.

*   **Environment Variables:** Standard, especially in containerized environments (12-factor app). Use `os.Getenv` or libraries like `github.com/kelseyhightower/envconfig`.
    ```go
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080" // Default value
    }
    dbURL := os.Getenv("DATABASE_URL")
    ```
*   **Command-Line Flags:** Use the standard `flag` package for simple configurations.
    ```go
    var portFlag = flag.String("port", "8080", "Port to listen on")
    flag.Parse() // Parse flags after defining them
    fmt.Println("Listening on port:", *portFlag)
    ```
*   **Configuration Files:** (JSON, YAML, TOML) Good for more complex configurations. Often combined with env vars/flags for overrides. Popular libraries:
    *   `spf13/viper`: Feature-rich, supports multiple formats, env/flag binding, watching changes.
    *   `gopkg.in/yaml.v3`, `encoding/json`, etc. for direct parsing.
*   **Precedence:** Often desirable: Flags > Env Vars > Config File > Defaults. Libraries like Viper handle this.

### 6. Running in Production - Operational Concerns

*   **Graceful Shutdown:** Allow the server to finish ongoing requests before shutting down when it receives a signal (like `SIGINT` (Ctrl+C) or `SIGTERM` (from orchestrators)).
    ```go
    package main

    import (
        // ... other imports
        "context"
        "net/http"
        "os"
        "os/signal"
        "syscall"
        "time"
        "log"
    )

    func main() {
        // ... Setup router (mux) ...
        mux := http.NewServeMux()
        mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) { /*...*/ })

        server := &http.Server{
            Addr:    ":8080",
            Handler: mux,
        }

        // Run server in a goroutine so it doesn't block
        go func() {
            log.Println("Server starting on :8080")
            if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
                log.Fatalf("ListenAndServe failed: %v", err)
            }
        }()

        // Wait for interrupt signal to gracefully shutdown the server
        quit := make(chan os.Signal, 1)
        signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
        <-quit // Block until signal is received
        log.Println("Shutting down server...")

        // Context with timeout to force shutdown if graceful period exceeded
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        if err := server.Shutdown(ctx); err != nil {
            log.Fatal("Server forced to shutdown:", err)
        }

        log.Println("Server exiting")
    }
    ```
*   **Health Checks:** Expose endpoints (e.g., `/healthz`, `/readyz`) for load balancers and orchestrators (like Kubernetes) to check application status.
    *   `/healthz`: Basic check (is the process running?).
    *   `/readyz`: Deeper check (is the app ready to serve traffic? e.g., DB connected?).
*   **Structured Logging:** Use libraries like `slog` (Go 1.21+), `zap`, `zerolog` to write logs in JSON or other machine-parsable formats. Log to `stdout`/`stderr` in containers; let the container runtime/orchestrator handle log aggregation.
*   **Monitoring/Metrics:** Expose application metrics (request counts, latency, error rates, goroutine count) typically via a `/metrics` endpoint in Prometheus format. Use libraries like `prometheus/client_golang`.

### 7. Basic CI/CD Pipeline Steps

Automate the build and deployment process.

1.  **Lint & Vet:** Run `golangci-lint run ./...` and `go vet ./...`.
2.  **Test:** Run `go test -race -cover ./...`.
3.  **Build:** Run `go build` (often cross-compiling with version info).
4.  **Build Docker Image:** Use `docker build`. Tag appropriately (e.g., with Git commit SHA and/or version tag).
5.  **Push Docker Image:** Push to a container registry (Docker Hub, ECR, GCR, etc.).
6.  **Deploy:** Update the service (e.g., Kubernetes deployment, ECS task definition) to use the new image version.

---

This cheatsheet provides a solid overview of common Go build and deployment practices. Remember that specific choices (like base images or configuration methods) depend on your project's requirements and infrastructure.
