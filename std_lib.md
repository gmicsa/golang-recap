## Go Standard Library Power-Ups (`io`, `time`, `encoding`) review

This cheatsheet highlights useful features, interfaces, and patterns within the `io`, `time`, and various `encoding` packages.

### 1. The `io` Package: Streams and Interfaces

The `io` package provides basic interfaces to I/O primitives. Its key strength lies in composability using interfaces like `Reader` and `Writer`.

*   **Core Interfaces:**
    *   `io.Reader`: Represents the read end of a stream.
        ```go
        type Reader interface {
            Read(p []byte) (n int, err error) 
        } 
        // Reads up to len(p) bytes into p. Returns bytes read and error (io.EOF on end).
        ```
    *   `io.Writer`: Represents the write end of a stream.
        ```go
        type Writer interface {
            Write(p []byte) (n int, err error)
        }
        // Writes len(p) bytes from p. Returns bytes written and error.
        ```
    *   `io.Closer`: Represents the ability to close a stream/resource.
        ```go
        type Closer interface {
            Close() error
        }
        // Always check the error from Close()! Often used with defer.
        ```
    *   Combined: `io.ReadCloser`, `io.WriteCloser`, `io.ReadWriteCloser`, `io.ReadSeeker`, etc.
*   **Essential Helpers:**
    *   `io.Copy(dst Writer, src Reader)`: Copies from `src` to `dst` until EOF or error. Efficient. Returns bytes copied and error.
        ```go
        bytesCopied, err := io.Copy(os.Stdout, fileReader) 
        ```
    *   `io.ReadAll(r Reader)`: Reads everything from `r` until EOF or error. Returns `[]byte` and error. **Use with caution** on untrusted/large inputs (can exhaust memory).
        ```go
        bodyBytes, err := io.ReadAll(httpResp.Body) 
        ```
    *   `io.WriteString(w Writer, s string)`: Writes a string to a writer.
    *   `io.Discard`: An `io.Writer` that discards all writes. Useful for testing or consuming unwanted output.
        ```go
        io.Copy(io.Discard, someReader) // Consume and discard data
        ```
*   **In-Memory `io`:**
    *   `bytes.Buffer`: Implements `io.Reader`, `io.Writer`, `io.ByteReader`, `io.ByteWriter`. A variable-sized buffer of bytes. Excellent for composing data in memory before writing, or for testing.
        ```go
        var buf bytes.Buffer
        buf.WriteString("Hello, ")
        fmt.Fprintf(&buf, "World %d!", 2023) 
        content := buf.String() // "Hello, World 2023!"
        io.Copy(os.Stdout, &buf) // Read from the buffer
        ```
    *   `strings.Reader`: Implements `io.Reader`, `io.ReaderAt`, `io.Seeker` on a string. Useful for treating a string as a readable stream.
        ```go
        r := strings.NewReader("Some input string")
        io.Copy(os.Stdout, r)
        ```
*   **Composition Helpers:**
    *   `io.MultiWriter(writers ...Writer)`: Returns a writer that duplicates writes to all provided writers.
        ```go
        // Write to both stdout and a file simultaneously
        f, _ := os.Create("log.txt")
        defer f.Close()
        mw := io.MultiWriter(os.Stdout, f)
        mw.Write([]byte("This goes everywhere.\n"))
        ```
    *   `io.MultiReader(readers ...Reader)`: Returns a reader that reads sequentially from the provided readers until all return EOF.
    *   `io.Pipe()`: Creates a synchronous in-memory pipe (`*PipeReader`, `*PipeWriter`). Writes to the writer block until read from the reader. Useful for connecting components concurrently. **Must close writer** to signal EOF to reader.
        ```go
        pr, pw := io.Pipe()
        go func() {
            defer pw.Close() // VERY IMPORTANT
            fmt.Fprintln(pw, "Data via pipe")
        }()
        io.Copy(os.Stdout, pr) // Reads "Data via pipe\n"
        ```
    *   `io.LimitedReader(r Reader, n int64)`: Returns a reader that reads from `r` but stops after `n` bytes. Useful for preventing DoS or enforcing size limits.

### 2. The `time` Package: Handling Time & Duration

Provides functionality for measuring and displaying time.

*   **Core Types:**
    *   `time.Time`: Represents an instant in time with nanosecond precision. Includes location (time zone).
    *   `time.Duration`: Represents elapsed time between two instants (int64 nanoseconds).
*   **Getting Time:**
    *   `time.Now()`: Current local time.
    *   `time.Date(year, month, day, hour, min, sec, nsec, loc)`: Construct a specific time. `time.UTC` and `time.Local` are predefined locations.
*   **Formatting & Parsing:** **Key Concept:** Use the specific reference time `Mon Jan 2 15:04:05 MST 2006` (`2006-01-02T15:04:05Z07:00`) to define layouts.
    *   `t.Format(layout string)`: Format time `t` according to the `layout`.
    *   `time.Parse(layout, value string)`: Parse `value` string according to `layout`. Returns `time.Time` and error.
    *   `time.ParseInLocation(layout, value string, loc *Location)`: Parse using a specific location.
    *   **Predefined Layouts:** `time.RFC3339`, `time.RFC1123`, `time.Kitchen`, etc.
        ```go
        now := time.Now()
        // Formatting
        fmt.Println(now.Format(time.RFC3339)) // "2023-10-27T10:30:00+01:00" (example)
        fmt.Println(now.Format("2006-01-02 15:04")) // "2023-10-27 10:30" 

        // Parsing
        t, err := time.Parse(time.RFC3339, "2023-11-01T12:00:00Z")
        if err != nil { /* handle error */ }
        fmt.Println(t.Year()) // 2023
        ```
*   **Time Zones:**
    *   `time.LoadLocation(name string)`: Load a location (e.g., "America/New_York"). Requires timezone database on system.
    *   `t.In(loc *Location)`: Convert time `t` to a different location.
    *   `t.UTC()`: Convert to UTC.
    *   `t.Local()`: Convert to system's local time.
*   **Durations & Calculations:**
    *   Constants: `time.Nanosecond`, `time.Microsecond`, `time.Millisecond`, `time.Second`, `time.Minute`, `time.Hour`.
    *   `time.ParseDuration(s string)`: Parse duration string (e.g., "1h30m", "5s", "1.5h").
    *   `t.Add(d Duration)`: Add duration `d` to time `t`.
    *   `t.Sub(u Time)`: Calculate duration between `t` and `u` (`t - u`).
    *   `t.AddDate(years, months, days)`: Add years, months, days. Handles calendar complexities.
    *   Comparisons: `t.Before(u)`, `t.After(u)`, `t.Equal(u)`.
*   **Timers & Tickers:**
    *   `time.Sleep(d Duration)`: Pause current goroutine for duration `d`.
    *   `time.After(d Duration)`: Returns a channel (`<-chan Time`) that receives the current time after duration `d`. Useful in `select`.
    *   `time.NewTimer(d Duration)`: Creates a `*Timer` that sends the current time on its channel `C` after duration `d`. Can be stopped (`Stop()`) or reset (`Reset()`). **Important:** Must ensure channel is drained if `Stop` returns `false`.
    *   `time.NewTicker(d Duration)`: Creates a `*Ticker` that sends the current time on its channel `C` repeatedly at intervals of duration `d`. Must be stopped (`Stop()`) to release resources.
        ```go
        timer := time.NewTimer(2 * time.Second)
        ticker := time.NewTicker(500 * time.Millisecond)
        defer ticker.Stop()

        select {
        case <-timer.C:
            fmt.Println("Timer expired!")
        case t := <-ticker.C:
            fmt.Println("Tick at", t)
        // case <- ctx.Done(): // Integrate with context cancellation
        }
        ```
*   **Monotonic Time:** Go internally uses monotonic time (unaffected by system clock adjustments) when calculating durations (`Sub`, `Since`). `time.Time` values store both "wall clock" and monotonic readings where available.

### 3. `encoding/*` Packages: Data Conversion

Handles conversion between Go types and byte representations (JSON, XML, Gob, Base64, etc.).

*   **`encoding/json`:** Most common for web APIs.
    *   `json.Marshal(v any)`: Encode Go value `v` into JSON `[]byte`.
    *   `json.Unmarshal(data []byte, v any)`: Decode JSON `data` into Go value `v` (must be a pointer).
    *   **Struct Tags:** Control encoding/decoding using `json:"..."` tags.
        ```go
        type User struct {
            ID   int    `json:"id"`
            Name string `json:"name,omitempty"` // Omit if zero value
            Pass string `json:"-"`             // Ignore this field
        }
        ```
    *   **Streaming (`json.Encoder`/`Decoder`):** Efficient for streams (network, files). Avoids loading entire data into memory.
        ```go
        // Decoding from io.Reader (e.g., http.Request.Body)
        var user User
        err := json.NewDecoder(r.Body).Decode(&user) 
        
        // Encoding to io.Writer (e.g., http.ResponseWriter)
        resp := map[string]string{"status": "ok"}
        err = json.NewEncoder(w).Encode(resp) 
        ```
    *   **Custom Marshaling:** Implement `json.Marshaler` / `json.Unmarshaler` interfaces for custom logic.
    *   **Arbitrary JSON:** Use `map[string]interface{}` or `[]interface{}`. `json.RawMessage` can delay decoding of a field.
*   **`encoding/gob`:** Go-specific binary encoding. Faster and more compact than JSON for Go-to-Go communication (RPC, caching). Not cross-language compatible. Streams using `gob.Encoder`/`gob.Decoder`. Requires registering types exchanged (`gob.Register`).
*   **`encoding/binary`:** For reading/writing fixed-size binary data (numbers). Specify byte order (`binary.BigEndian`, `binary.LittleEndian`). Useful for custom network protocols or file formats.
    ```go
    var pi float64 = math.Pi
    buf := new(bytes.Buffer)
    err := binary.Write(buf, binary.LittleEndian, pi)
    
    var decodedPi float64
    err = binary.Read(buf, binary.LittleEndian, &decodedPi)
    ```
*   **`encoding/xml`:** Similar structure to `encoding/json` (Marshal, Unmarshal, Encoder, Decoder, struct tags `xml:"..."`). More complex due to XML's nature.
*   **`encoding/base64`:** Encodes/decodes binary data to base64 text representation. Various standard encodings (`StdEncoding`, `URLEncoding`). Uses streaming encoders/decoders.
*   **`encoding/hex`:** Encodes/decodes binary data to hexadecimal text representation.

---

These packages are workhorses in Go development. Understanding their interfaces and helpers allows for writing flexible, efficient, and idiomatic Go code. Remember to always check for errors returned by these functions!
