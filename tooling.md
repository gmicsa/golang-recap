## Go Tooling & Modules review

This cheatsheet covers the standard Go command-line tools and the Go Modules system for dependency management.

### 1. Common `go` Commands

*   **`go build [packages]`:** Compiles packages and dependencies.
    *   `go build`: Builds the package in the current directory into an executable (if `main` package).
    *   `go build ./...`: Builds all packages below the current directory.
    *   `go build -o myapp`: Builds the current package and outputs the executable as `myapp`.
    *   `go build -v ./...`: Verbose output, showing packages as they are compiled.
    *   `go build -tags="tag1,tag2"`: Include files matching build tags (see Section 4).
    *   `go build -ldflags="..."`: Pass arguments to the linker (see Section 4).
    *   `go build -race`: Build with the race detector enabled (adds performance overhead).
*   **`go run [files or packages]`:** Compiles and runs the specified main package. Good for quick testing.
    *   `go run main.go`: Compiles and runs `main.go`.
    *   `go run .`: Compiles and runs the package in the current directory.
    *   `go run -race main.go`: Run with the race detector.
*   **`go test [packages]`:** Runs tests and benchmarks.
    *   `go test`: Runs tests in the current directory's package.
    *   `go test ./...`: Runs tests in all packages below the current directory.
    *   `go test -v`: Verbose output, including test names and status.
    *   `go test -run TestMyFunc`: Run tests matching the regex `TestMyFunc`.
    *   `go test -bench=.`: Run benchmarks (see Performance Cheatsheet).
    *   `go test -cover`: Show test coverage percentage.
    *   `go test -coverprofile=coverage.out`: Generate a coverage profile file.
    *   `go test -race`: Run tests with the race detector.
    *   `go test -count=1`: Disable test result caching.
*   **`go fmt [packages]`:** Formats Go source code according to `gofmt` standards.
    *   `go fmt ./...`: Format all Go files below the current directory. **Run this often!**
*   **`go vet [packages]`:** Examines Go source code and reports suspicious constructs (potential bugs).
    *   `go vet ./...`: Run vet on all packages below the current directory. **Include in CI!**
*   **`go install [packages]`:** Compiles and installs packages or commands.
    *   `go install`: Compiles and installs the command from the current directory into `$GOPATH/bin` or `$GOBIN`.
    *   `go install example.com/cmd/mytool@latest`: Downloads, compiles, and installs a specific tool version. Preferred way to install Go tools since Go 1.16+.
*   **`go get [package@version]`:** (Primarily for Tools/Executables now) Adds dependencies to `go.mod` or installs executables.
    *   **Since Go 1.17+, `go get` should NOT be used for adding/updating dependencies in your module.** Use `go mod tidy` or edit `go.mod` directly.
    *   `go get example.com/cmd/mytool`: Downloads and installs a tool (use `go install ...@latest` instead).
*   **`go clean [flags]`:** Removes object files and cached files.
    *   `go clean -modcache`: Remove the entire module download cache (use if cache seems corrupt).
    *   `go clean -testcache`: Remove test results cache.
*   **`go list [flags] [packages]`:** Lists information about packages.
    *   `go list -m all`: List the current module and all its dependencies.
    *   `go list -json ./...`: Output detailed package info in JSON format.

### 2. Go Modules (`go mod`)

The standard dependency management system. Enabled by default outside `$GOPATH`.

*   **`go mod init [module-path]`:** Initializes a new module in the current directory. Creates `go.mod`.
    *   `go mod init example.com/myproject`
*   **`go mod tidy`:** **Crucial command.** Ensures `go.mod` matches source code.
    *   Adds any missing module requirements needed for build/test.
    *   Removes unused module requirements.
    *   Updates `go.sum` with checksums for direct and indirect dependencies. **Run this before committing changes!**
*   **`go mod download [module@version]`:** Downloads specified modules (or all dependencies if none specified) to the local cache.
*   **`go mod vendor`:** Creates a `vendor` directory containing copies of all dependencies.
    *   Allows hermetic builds (builds that don't rely on network access).
    *   Use `go build -mod=vendor` to force the build to use the `vendor` directory.
*   **`go mod why [packages]`:** Shows the shortest path explaining why packages are included as dependencies.
    *   `go mod why golang.org/x/text`
*   **`go mod graph`:** Prints the module requirement graph (often large).
*   **`go mod edit [flags]`:** Edits the `go.mod` file. Often used for `replace` directives.
    *   `go mod edit -replace=example.com/original/module=/path/to/local/fork`
    *   `go mod edit -go=1.19`: Sets the Go language version directive.
    *   `go mod edit -module=example.com/new/path`: Changes the module path (rare).
*   **`go mod verify`:** Checks that dependencies in the module cache haven't been modified since download.

### 3. Module Files

*   **`go.mod`:** Defines the module.
    *   `module [module-path]`: Declares the module's path (e.g., `example.com/myproject`).
    *   `go [version]`: Specifies the minimum Go language version required (e.g., `go 1.19`).
    *   `require`: Lists direct dependencies and their minimum required versions (follows Semantic Versioning).
        *   `require example.com/dependency v1.2.3`
        *   `require indirect.dependency v0.5.0 // indirect`: Indicates an indirect dependency (dependency of a dependency).
    *   `replace`: Replaces a required module version with a different version or a local path. Useful for forks or local development.
        *   `replace example.com/original => example.com/fork v1.1.5`
        *   `replace example.com/original => ../local-version`
    *   `exclude`: Excludes a specific version of a dependency (use sparingly, `replace` is often better).
*   **`go.sum`:** Contains expected cryptographic checksums (hashes) of module content.
    *   Ensures dependencies haven't been tampered with (integrity).
    *   Managed automatically by commands like `go mod tidy`, `go get`, `go build`.
    *   **Should be committed to version control** alongside `go.mod`.

### 4. Workspaces (`go work`) (Go 1.18+)

Manages working across *multiple* related modules simultaneously. Useful for making changes in a dependency and testing them in a main module without needing `replace` directives in each `go.mod`.

*   **`go work init [mod-dirs...]`:** Initializes a workspace in the current directory, creating `go.work`.
    *   `go work init ./main-app ./lib-dependency`
*   **`go work use [mod-dir]`:** Adds a module path to the `go.work` file.
    *   `go work use ../another-lib`
*   **`go work sync`:** Syncs build dependencies from workspace modules into `go.work`.
*   **`go.work` File:**
    *   `go [version]`: Go language version.
    *   `use`: Lists paths to modules included in the workspace (relative or absolute).
        *   `use ./moduleA`
        *   `use ../../libs/moduleB`
    *   `replace`: Similar to `go.mod`, but applies workspace-wide. Takes precedence over `go.mod` replaces.

### 5. Build Configuration & Tags

*   **Build Constraints (Build Tags):** Conditional compilation using comments.
    *   Syntax: `//go:build [build constraint]` (Go 1.17+)
    *   Must be followed by a blank line.
    *   Can appear at the top of any Go source file (not just `_test.go`).
    *   Logic:
        *   Space-separated = OR (`//go:build linux darwin`)
        *   Comma-separated = AND (`//go:build linux,amd64`)
        *   `!` = NOT (`//go:build !windows`)
    *   Activating: `go build -tags="tag1,tag2"`
    *   Common Tags: OS (`linux`, `windows`, `darwin`), Arch (`amd64`, `arm64`), `debug`, `integration`.
*   **Cross-Compilation:** Build for different OS/Architecture. Set environment variables:
    *   `GOOS`: Target Operating System (e.g., `linux`, `windows`, `darwin`).
    *   `GOARCH`: Target Architecture (e.g., `amd64`, `arm64`, `386`).
    *   Example: `GOOS=linux GOARCH=amd64 go build -o myapp-linux .`
*   **Linker Flags (`-ldflags`):** Pass flags to the linker during build.
    *   **Strip Symbols/Debug Info:** Reduce binary size (`-s -w`).
        *   `go build -ldflags="-s -w" .`
    *   **Set Variable Value:** Embed version info, commit hash, etc., at build time.
        *   Go code: `var version = "dev"`
        *   Build command: `go build -ldflags="-X 'main.version=1.2.3'" .` (Note the single/double quotes and full path `main.version`).

### 6. Static Analysis & Linting

*   **`go vet`:** Built-in tool for finding suspicious constructs. Run it!
*   **`golangci-lint`:** (Third-party, highly recommended) Aggregates many different linters, fast, configurable.
    *   Install: `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest`
    *   Run: `golangci-lint run ./...`
    *   Configuration: `.golangci.yml` file.

### 7. Debugging

*   **`delve`:** The standard debugger for Go.
    *   Install: `go install github.com/go-delve/delve/cmd/dlv@latest`
    *   Debug executable: `dlv debug main.go` or `dlv exec ./myapp`
    *   Debug tests: `dlv test ./...`
    *   Attach to running process: `dlv attach <pid>`
    *   Common commands: `break <loc>`, `continue`, `next`, `step`, `print <var>`, `list`, `goroutines`, `bt` (backtrace).

---

Mastering these tools is fundamental to efficient Go development. Always refer to the official Go documentation (`go help <command>`) for the most detailed and up-to-date information.
