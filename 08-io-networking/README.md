# Phase 08 — I/O & Networking

> **Mục tiêu:** Hiểu request thực sự đi từ client → network → server như thế nào. Không cần build Netty, nhưng phải hiểu tại sao Netty tồn tại.

---

## Part 1 — I/O Streams

### Byte Streams (binary)
```
InputStream / OutputStream  ← base abstract
    ├── FileInputStream / FileOutputStream      ← file
    ├── ByteArrayInputStream                    ← memory
    └── BufferedInputStream / BufferedOutputStream ← buffering (performance)
```

### Character Streams (text)
```
Reader / Writer             ← base abstract
    ├── FileReader / FileWriter
    ├── BufferedReader / BufferedWriter         ← buffering
    └── InputStreamReader / OutputStreamWriter  ← bridge byte ↔ char
```

**Luôn dùng try-with-resources:**
```java
try (var reader = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) { process(line); }
}
```

**Tại sao Buffered quan trọng:**
- Mỗi `read()`/`write()` không có buffer = system call
- System call tốn vài microseconds
- Buffer gộp nhiều operations → 100x+ speedup cho small reads

---

## Part 2 — NIO (Non-blocking I/O)

### Core concepts
```
Channel     ← bi-directional I/O (thay InputStream/OutputStream)
Buffer      ← data container (flip, clear, compact)
Selector    ← multiplexing: 1 thread, nhiều Channels
```

### Buffer lifecycle
```
Allocate → Write (flip) → Read (clear/compact) → Write again
```
```java
ByteBuffer buf = ByteBuffer.allocate(1024);
channel.read(buf);   // write mode
buf.flip();          // switch to read mode
while (buf.hasRemaining()) {
    process(buf.get());
}
buf.clear();         // ready to write again
```

### Selector (non-blocking server)
```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // block until at least one channel ready
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) { handleAccept(key); }
        if (key.isReadable())   { handleRead(key); }
    }
}
```

**1 thread xử lý N connections** — đây là model của Netty, Tomcat NIO connector.

### NIO.2 — File API
```java
Path path = Path.of("/data/file.txt");

// Read/Write
Files.readAllLines(path);
Files.readString(path);
Files.writeString(path, content, StandardOpenOption.APPEND);
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);

// Walk directory
Files.walk(dir)
    .filter(p -> p.toString().endsWith(".java"))
    .forEach(System.out::println);

// Watch for changes
WatchService watcher = FileSystems.getDefault().newWatchService();
dir.register(watcher, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);
```

---

## Part 3 — Networking

### TCP vs UDP
| | TCP | UDP |
|--|-----|-----|
| Connection | Yes (3-way handshake) | No |
| Reliability | Guaranteed delivery, ordering | Best-effort |
| Speed | Slower (overhead) | Faster |
| Dùng khi | HTTP, database, file transfer | DNS, video streaming, gaming |

### TCP Socket (Java)
```java
// Server
ServerSocket server = new ServerSocket(8080);
Socket client = server.accept();  // blocking
InputStream in = client.getInputStream();

// Client
Socket socket = new Socket("localhost", 8080);
OutputStream out = socket.getOutputStream();
```

### HTTP Concepts

**Request structure:**
```
POST /api/orders HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer <token>

{"productId": 1, "quantity": 2}
```

**Connection: Keep-Alive** — reuse TCP connection cho nhiều requests (HTTP/1.1 default).

**TLS handshake** (HTTPS):
```
Client Hello → Server Hello → Certificate → Key Exchange → Application Data
```

### Java HTTP Client (Java 11+)
```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Authorization", "Bearer " + token)
    .GET()
    .build();

// Sync
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());

// Async
CompletableFuture<HttpResponse<String>> future =
    client.sendAsync(request, BodyHandlers.ofString());
```

---

## Part 4 — Serialization

### JSON (Jackson)
```java
ObjectMapper mapper = new ObjectMapper();

// Serialize
String json = mapper.writeValueAsString(object);

// Deserialize
MyClass obj = mapper.readValue(json, MyClass.class);

// With configuration
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

### Khi nào dùng gì
| Format | Khi nào | Trade-off |
|--------|---------|-----------|
| JSON | REST API, config | Human readable, verbose |
| XML | Legacy, SOAP | Verbose, namespaces |
| Protobuf | High-performance RPC, Kafka | Binary, schema required, smaller/faster |
| Avro | Kafka, Hadoop | Schema evolution |

---

## Part 5 — Blocking vs Non-blocking I/O

```
Blocking I/O:
Thread → read() → [WAIT] → data arrives → continue
(thread blocked, cannot do other work)

Non-blocking I/O:
Thread → register interest → [do other work]
         ← event notification ← data arrives
(thread free while waiting)
```

**Implication cho server:**
- Blocking: 1 thread per connection → 10,000 connections = 10,000 threads → OOM
- Non-blocking: 1 thread nhiều connections → Netty, NIO Selector model
- Virtual Threads (Java 21): blocking API but non-blocking runtime → best of both

---

## Project — Mini HTTP Client/Server

**Không dùng Spring, không dùng Netty. Chỉ Java NIO/Socket thuần.**

```
src/
├── server/
│   ├── HttpServer.java           ← NIO Selector-based single-threaded server
│   ├── HttpRequest.java          (Record: method, path, headers, body)
│   ├── HttpResponse.java         (Record: status, headers, body)
│   ├── RequestParser.java        ← parse raw bytes → HttpRequest
│   └── Router.java               ← path → handler mapping
├── client/
│   ├── HttpClient.java           ← java.net.http.HttpClient wrapper
│   └── RetryClient.java          ← retry with exponential backoff
├── file/
│   ├── FileWatcher.java          ← WatchService demo
│   └── FileCopyBenchmark.java    ← Stream vs NIO channel comparison
└── Main.java
```

**Yêu cầu:**
1. Server nhận GET/POST requests, trả về JSON responses
2. Support multiple concurrent connections (non-blocking NIO)
3. Client với retry và timeout
4. Benchmark: blocking socket vs NIO cho 1000 concurrent connections

---

## Checklist — 6 câu hỏi

Ví dụ với `NIO Selector`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Multiplexer — 1 thread giám sát nhiều channels cho I/O readiness |
| 2 | Giải quyết gì? | Tránh 1-thread-per-connection model, scale đến hàng nghìn connections |
| 3 | Hoạt động thế nào? | Register channel + interest ops; `select()` blocks; iterate SelectedKeys |
| 4 | Khi nào dùng? | High-concurrency servers, custom protocol; lý do Netty/Tomcat dùng |
| 5 | Khi nào không? | Java 21+ virtual threads đơn giản hơn cho hầu hết use cases |
| 6 | Debug thế nào? | `selector.keys().size()` để đếm registered channels; thread dump nếu stuck |

---

## Run

```bash
mvn compile -pl phase-08-io-networking
mvn exec:java -pl phase-08-io-networking \
    -Dexec.mainClass="com.nguyenngoc.phase08.Main"
```
