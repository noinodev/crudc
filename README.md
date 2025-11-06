# High-Performance REST API in C 'CRUDC'

Multi-threaded HTTP REST API achieving 300k+ requests/second with PostgreSQL integration, 
custom authentication, and binary file uploads.

## Architecture

**Thread Model:** Single-producer-multiple-consumer (SPMC) worker pool (similar to nginx/apache)
- Asynchronous worker threads with FIFO circular queue for O(1) task scheduling
- Shared locking queue with thread-safe dequeuing
- Minimal heap allocation (stack-preferred) for faster memory access
- API key session caching

## Features
- User authentication with API tokens
- PostgreSQL database with optimized schema (bit-packing, MD5-based file storage)
- Binary file upload with streaming support
- Parameterized SQL queries (SQL injection prevention)
- Cross-platform (POSIX, Windows via MinGW-64)

## Why C?
Written in C to maximize performance and minimize resource consumption. Stack-preferred allocation and embedded-grade libraries enable deployment in resource-constrained environments while achieving high throughput.

## Performance Highlights
- Peak throughput: 350,000 requests/second (localhost)
- Optimal configuration: 30,000 sustained req/s with 2 worker threads
- Unloaded latency: 0.9ms average
- Loaded latency: 112.8ms average (10 worker threads under max load)

## Tech Stack
- **HTTP Parsing:** picohttpparser (embedded systems-grade)
- **JSON:** tiny-json (stack-only)
- **Database:** PostgreSQL via libpq
- **Security:** OpenSSL (MD5 hashing, authentication)

## API Endpoints
- `POST /auth/create` - Create user account
- `POST /auth/login` - Authenticate and retrieve API token
- `POST /api/sample` - Upload sample metadata
- `POST /api/file` - Upload binary files (PNG)

## Build and run instructions
Windows with MinGW:
 - install openssl and libpq dependencies
 - cd into root folder
 - run 'mingw32-make server'
 - run '.\bin\server [thread count]'

tests/ instructions
integration.c - server load testing
 - run .\bin\server [thread count]
 - gcc compile integration.c (windows requires lws2_32 dependency)
 - run compiled executable '.\a.exe [thread count] [socket count] [packet count]'

time.c - server latency testing
 - run .\bin\server [thread count]
 - gcc compile time.c (windows requires lws2_32 dependency)
 - run compiled executable '.\a.exe [packet count]'

# CRUDC - Performance Testing

## 1.0 Introduction and Strategy

The requirements of the API are simple: interact with a database securely and quickly. This API was written in C to leverage its performance; therefore, the best metric of testing is HTTP request throughput. The server is designed to be scalable, running on any n threads and connecting to any database, with additional security features like user authentication.

## 2.0 Integration / Performance Testing

### 2.1 integration.c

Integration testing simulates deployment and assesses real-world performance. A custom load testing client was created to distribute HTTP requests and stress-test the server. Details:

- Executes from command line with 3 arguments: number of threads, sockets per thread, and packets per socket
- For every thread/socket/packet, randomly selects from 3 request types: create account, login, and upload sample
- Each request has randomized parameters to test various API cases
- Packets sent in round-robin fashion per thread to minimize TCP concatenation and simulate more file descriptors
- Visualizes connection status, profiles packet send times, and tracks send() success rate

### 2.2 time.c

A second client tests latency, HTTP edge cases, and streaming protocol accuracy:

- Executes from command line with 1 argument: number of packets to send
- Sends randomized valid and invalid HTTP requests
- Packets sent at 100ms intervals to prevent TCP concatenation
- Can run in parallel with integration.c to test response times under load

---

## Load Testing Results

### Test 1: Single-threaded Client, Multithreaded Server

**Configuration:**
- Server: 4 worker threads
- Client: 1 worker thread, 500 sockets/thread, 100,000 packets/socket

**Results:**
- **25,000 requests/second** average
- Server processing packets faster than client could send them

**Refinement:** Improve client output speed with multithreading

---

### Test 2: Multithreaded Client/Server Load Test

**Configuration:**
- Server: 4 worker threads
- Client: 5 worker threads, 100 sockets/thread, 62,350 packets/socket
- Total: 30,000,000 packets

**Results:**
- **350,000 requests/second** average
- Apparent error with file descriptor handling and packet concatenation
- High rate of transmission failure

**Refinement:** Implement packet streaming on server side to process concatenated packets in a single task

---

### Test 3: Single-threaded Packet Streaming Test

**Configuration:**
- Server: 1 worker thread
- Client: 1 worker thread, 1 socket/thread, 100,000 packets/socket

**Results:**
- **30,000 requests/second** average
- Increased single-threaded performance
- Significantly reduced transmission failure
- Noted file descriptor handling issues

**Refinement:** Implement socket timeout and larger memory pool (500 clients + 1000 tasks → 10,000 clients + 10,000 tasks)

---

### Test 4: Integration Test with Memory Pool

Tested socket timeout implementation, larger memory pool, and realistic scaling with comprehensive profiling (100s of nanoseconds resolution).

#### 1 Worker Thread
**Configuration:**
- Server: 1 worker thread
- Client: 2 worker threads, 500 sockets/thread, 5,000 packets/socket

**Results:**
- Clients: 1,000
- Tasks: 80,779 [db: 31, login: 1,114, create: 1,140]
- **Average task time: 0.012407ms**
- Average time per client: 0.962ms
- Server load: 96.21%

**Observation:** High parallelism appears to negatively correlate with performance, possibly due to cache locality. Solvable with work-stealing queues. Atomic variables for profiling add significant overhead.

#### 4 Worker Threads
**Configuration:**
- Server: 4 worker threads
- Client: 2 worker threads, 500 sockets/thread, 5,000 packets/socket

**Results:**
- Clients: 1,000
- Tasks: 35,607 [db: 1,005, login: 3,761, create: 3,760]
- **Average task time: 0.144178ms**
- Average time per client: 3.812ms
- Server load: 95.39%

#### 2 Worker Threads (Optimal)
**Configuration:**
- Server: 2 worker threads
- Client: 2 worker threads, 500 sockets/thread, 5,000 packets/socket

**Results:**
- Clients: 1,000
- Tasks: 52,673 [db: 255, login: 2,427, create: 2,361]
- **Average task time: 0.043335ms**
- Average time per client: 1.992ms
- Server load: 99.48%

---

## Latency Testing Results

### Test 1: Unloaded Latency

**Configuration:**
- Server: idle, 2 worker threads
- Client: 100 packets

**Results:**
- **Average response time: 0.9ms**

---

### Test 2: Loaded Latency (1 Worker Thread)

**Configuration:**
- Server: 1 worker thread, loaded by integration.c
- Integration.c: 4 threads, 250 sockets/thread, 500,000 packets/socket
- Client: 10 packets

**Results:**
- **Average response time: 184.6ms**
- **20,000 tasks/second** average

**Observation:** Maximum server load greatly increases response time. Likely caused by packet streaming holding worker threads. Parallelization is vital for high throughput under load.

---

### Test 3: Loaded Latency (10 Worker Threads)

**Configuration:**
- Server: 10 worker threads, loaded by integration.c
- Integration.c: 4 threads, 250 sockets/thread, 500,000 packets/socket
- Client: 100 packets

**Results:**
- **Average response time: 112.8ms**
- **30,000 tasks/second** average

**Observation:** Parallelization allows socket streaming and expensive tasks like image upload to complete efficiently while maintaining low latency for trivial requests.

---

### Test 4: HTTP Parsing Test

**Configuration:**
- Client: 1,000 packets

**Results:**
- No abnormalities for normal use
- Parsing abnormality with dummy invalid packet (returns '400 Bad Request' with incorrect JSON parsing error)

**Refinement:** Explore more edge cases in HTTP parsing and/or refactor for more modular parsing strategy

---

## 3.0 Observations and Future Optimization Strategy

The core functional requirement of 500 requests per minute has been greatly exceeded. The API is flexible, scalable, and cross-platform compatible. It uses low-impact dependencies and programming techniques to minimize resource consumption.

### Potential Optimizations:

**Improved caching mechanisms for database requests:**
- Database queries become expensive as the database grows
- Cache database hits/misses and check incoming data against cache
- Current authorization key cache uses O(n) search; should be replaced with hash table

**Improved task scheduling:**
- Current system uses mutual exclusion and circular buffer
- Should distribute tasks to private per-thread queues
- Implement work-stealing when threads become idle to balance load

**Improved testing environment:**
- Testing client and server on same machine introduces noise
- Harder to detect whether errors originate from client or server
- Shared system resources give less accurate benchmarks
- Should use separate system for client testing


