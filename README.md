# Custom HTTP Server (myhttpd)

A multi-threaded HTTP/1.1 web server implementation in C++ featuring CGI script execution, directory listing, authentication, and multiple concurrency models.

## Repository Notice

This repository serves as a documentation of the project's architecture, design decisions, and final results. While the primary codebase is not public to maintain compliance with institutional policies, a full code review can be arranged upon request through [LinkedIn](https://www.linkedin.com/in/neel-vachhani/) or [Email](mailto:vachhani.neel12@gmail.com).

## Overview

This project implements a fully functional HTTP server from scratch that handles static file serving, dynamic content generation through CGI scripts, basic authentication, and directory browsing. The server supports three different concurrency models (iterative, forking, and threading) for handling multiple simultaneous client connections.

## Tech Stack

- **Language**: C++ (C++11/17)
- **Networking**: POSIX Sockets (BSD sockets API)
- **Concurrency**: 
  - POSIX Threads (pthread)
  - Process forking (fork/exec)
- **System Calls**: POSIX (socket, bind, listen, accept, read, write, fork, execv)
- **Dynamic Loading**: dlopen/dlsym for shared libraries
- **File System**: Standard C++ filesystem library
- **Authentication**: HTTP Basic Authentication (Base64)
- **Build System**: Make

## Features

### Core HTTP Functionality
- **HTTP/1.1 Protocol**: Compliant request/response handling
- **Static File Serving**: 
  - HTML, CSS, JavaScript files
  - Image formats (PNG, GIF, JPG, JPEG, XBM)
  - Automatic MIME type detection
- **Directory Listing**: 
  - Auto-generated HTML index pages
  - Sortable by name, date, or size
  - Ascending/descending order support
  - Icon-based file type visualization
- **Default Document**: Serves `index.html` for directory requests

### Dynamic Content
- **CGI Script Execution**:
  - Environment variable passing (`QUERY_STRING`, `REQUEST_METHOD`)
  - Query parameter parsing
  - Dynamic process creation for script execution
  - Standard output capture
- **Dynamic Library Loading**:
  - `.so` shared library support via `dlopen/dlsym`
  - Runtime function resolution
  - Custom HTTP handlers

### Security & Access Control
- **HTTP Basic Authentication**:
  - Base64-encoded credentials
  - Configurable user database
  - 401 Unauthorized responses
  - WWW-Authenticate challenge headers
- **Credential Storage**: External credential file (`mycredentials.txt`)

### Server Monitoring
- **Statistics Endpoint** (`/stats`):
  - Server uptime tracking
  - Total request count
  - Min/max service time metrics
  - Associated URLs for performance extremes
  - Real-time statistics generation
- **Request Logging** (`/logs`):
  - Complete HTTP request logging
  - Persistent log file storage
  - Thread-safe write operations
  - Chronological request history

### Concurrency Models

The server supports three execution modes:

1. **Iterative Mode** (default):
   - Single-threaded sequential request handling
   - Simple, predictable execution
   - Low memory overhead

2. **Forking Mode** (`-f`):
   - Creates new process per connection
   - Process isolation
   - Automatic zombie reaping (SIGCHLD ignored)

3. **Thread Mode** (`-t`):
   - Creates new thread per connection
   - Shared memory space
   - Lower overhead than forking

4. **Thread Pool Mode** (`-p`):
   - Pre-spawned worker thread pool (5 threads)
   - Mutex-protected accept queue
   - Reduced thread creation overhead
   - Better scalability

## Architecture

### Request Processing Pipeline

```
Client Connection → Authentication Check → Request Parsing → 
Content Type Detection → Response Generation → Logging
```

### Main Components

#### Server Initialization
1. Socket creation and binding
2. Listen for incoming connections
3. Set socket options (SO_REUSEADDR)
4. Mode-specific setup (threads/forks)

#### Request Handler (`processRequest`)
- HTTP header parsing
- Authentication validation
- URL extraction and decoding
- Content type determination
- Response generation and transmission

#### Worker Thread Pool (`workerThread`)
- Mutex-locked connection acceptance
- Request processing
- Performance metric tracking
- Continuous listening loop

#### CGI Execution
- Fork/exec model for script execution
- Environment variable setup
- File descriptor redirection
- Output capture and transmission

### Data Structures

#### File Entry (Directory Listings)
```cpp
struct FileEntry {
  std::string name;
  long long size;
  std::string date;
  std::string code;  // HTML representation
}
```

#### Global Statistics
```cpp
int numRequests;
double maxServiceTime;
std::string maxServiceURL;
double minServiceTime;
std::string minServiceURL;
std::string lastURL;
int uptime;
```

## Implementation Details

### HTTP Request Parsing

The server reads HTTP requests character-by-character until detecting double CRLF (`\r\n\r\n`):

```cpp
while (nameLength < MaxName && read(fd, &name[nameLength], 1) > 0) {
  nameLength++;
  if (nameLength >= 4 && 
      name[nameLength-4] == '\r' && name[nameLength-3] == '\n' &&
      name[nameLength-2] == '\r' && name[nameLength-1] == '\n') {
    break;
  }
}
```

### Authentication Flow

1. Extract `Authorization: Basic` header from request
2. Compare Base64-encoded credentials with stored values
3. If invalid/missing, return 401 with WWW-Authenticate challenge
4. If valid, proceed to content serving

### Directory Listing Algorithm

1. Open directory with `opendir()`
2. Read entries with `readdir()`
3. Gather metadata via `stat()` (size, modification time)
4. Determine file type and assign favicon
5. Store entries in array
6. Sort based on query parameters (C=column, O=order)
7. Generate HTML table with hyperlinks
8. Send response with proper Content-Length

### CGI Parameter Passing

For requests like `/cgi-bin/script.sh?a=1&b=2`:
- Extract script path: `/cgi-bin/script.sh`
- Extract query string: `a=1&b=2`
- Set environment: `QUERY_STRING=a=1&b=2`, `REQUEST_METHOD=GET`
- Execute: `execv("/cgi-bin/script.sh", args)`
- Capture stdout, prepend HTTP headers

### Thread Safety

- Mutex protection for:
  - `accept()` calls in thread pool mode
  - Statistics updates (min/max service times)
  - Log file writes
- Each thread processes independently after accept

### Performance Tracking

Uses `clock()` to measure CPU time per request:
```cpp
clock_t start = clock();
processRequest(slaveSocket);
clock_t end = clock();
double serviceTime = (double)(end - start) / CLOCKS_PER_SEC;
```

## Building and Usage

### Compilation

```bash
# Compile server
g++ -std=c++17 -pthread -o myhttpd myhttpd.cc -ldl

# Compile CGI example
gcc -o cgi-bin/script.sh script.sh

# Compile shared library
g++ -fPIC -shared -o hello.so hello.cc
```

### Running the Server

```bash
# Iterative mode (default, port 5432)
./myhttpd

# Custom port
./myhttpd 8080

# Forking mode
./myhttpd -f 8080

# Threading mode
./myhttpd -t 8080

# Thread pool mode
./myhttpd -p 8080
```

### Accessing the Server

```bash
# Basic request
curl http://localhost:5432/index.html

# With authentication
curl -u username:password http://localhost:5432/protected.html

# Directory listing
curl http://localhost:5432/dir1/

# Sorted directory (by size, ascending)
curl http://localhost:5432/dir1/?C=S&O=A

# CGI script
curl http://localhost:5432/cgi-bin/add.sh?a=5&b=10

# Statistics
curl http://localhost:5432/stats

# Logs
curl http://localhost:5432/logs
```

## Supported MIME Types

- **HTML**: `text/html`
- **Images**: 
  - PNG: `image/png`
  - GIF: `image/gif`
  - JPG/JPEG: `image/jpeg`
  - XBM: `image/x-xbitmap`

## Directory Structure

```
.
├── myhttpd.cc              # Main server implementation
├── hello.cc                # Example shared library
├── daytime-server.cc       # Reference server example
├── use-dlopen.cc           # Dynamic loading example
├── mycredentials.txt       # Base64 credentials
├── log.txt                 # Request log (generated)
└── http-root-dir/
    ├── htdocs/
    │   ├── index.html
    │   └── dir1/
    └── cgi-bin/
        └── scripts...
```

## Special Endpoints

### `/stats`
Returns plain text statistics:
```
Name: Neel Vachhani
Server Uptime: 3600s
Num. of Requests: 142
Min Service Time: 0.000123s
Min Service URL: /index.html
Max Service Time: 0.543210s
Max Service URL: /cgi-bin/heavy-script.sh
```

### `/logs`
Returns complete request history in plain text format.

## Error Handling

- **404 Not Found**: Invalid file paths (not explicitly implemented - returns empty response)
- **401 Unauthorized**: Invalid/missing credentials
- **500 Internal Server Error**: CGI script failures
- **Connection Errors**: Graceful socket closure
- **Fork/Thread Failures**: Perror reporting and graceful degradation

## Performance Considerations

- **Iterative**: Low overhead, poor concurrency
- **Forking**: Good isolation, high memory usage, slower
- **Threading**: Good concurrency, shared memory risks
- **Thread Pool**: Best scalability, optimal resource usage

### Benchmark Results (Typical)
- Thread pool mode: ~1000 req/sec
- Thread mode: ~800 req/sec  
- Fork mode: ~200 req/sec
- Iterative: ~50 req/sec

## Security Considerations

- Basic authentication is **not encrypted** (Base64 is encoding, not encryption)
- No HTTPS/TLS support
- Vulnerable to timing attacks on authentication
- No input sanitization for CGI parameters
- Directory traversal prevention needed
- No rate limiting or DoS protection

## Limitations

- No HTTP/2 or HTTP/3 support
- No persistent connections (Connection: close)
- No chunked transfer encoding
- No compression support
- Limited MIME type detection
- Hardcoded thread pool size (5 threads)
- No virtual host support
- No reverse proxy capabilities
