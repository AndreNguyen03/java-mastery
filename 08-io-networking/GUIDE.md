# Phase 08 — I/O & Networking · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.1 Blocking vs Non-blocking I/O

**Blocking I/O (java.io):**
```
Thread A: read() ──────────────────────────── data arrives ──→ continue
          [blocked: doing nothing, holding OS thread]
```
1 connection = 1 thread. Server với 10K connections = 10K threads. Memory: 10K × 1MB = 10GB. Không scale.

**Non-blocking I/O (java.nio — Selector):**
```
Thread: select() ─→ [channel 1 ready] ─→ handle channel 1
               └──→ [channel 3 ready] ─→ handle channel 3
               └──→ [no events] ─→ do other work or wait
```
1 thread có thể quản lý N connections. Selector multiplex nhiều channels. Kernel thông báo khi IO sẵn sàng (epoll trên Linux, kqueue trên macOS).

**Trade-offs:**
| | Blocking (java.io) | Non-blocking (NIO) |
|---|---|---|
| Code | Simple, sequential | Complex, event-driven |
| Scalability | 1 thread per conn | Many conns per thread |
| Latency | Low (for single conn) | Similar |
| Use case | Few connections, simple protocol | High concurrency, event-driven |

**Với Virtual Threads (Java 21):** Blocking I/O + Virtual Threads = best of both worlds. Write simple blocking code, JVM makes it non-blocking under the hood. NIO Selector vẫn có vai trò trong network servers cần tùy chỉnh cao.

---

### 1.2 NIO Core Concepts

**ByteBuffer — State machine:**
```
capacity: 10 (total size)
position: con trỏ hiện tại
limit:    giới hạn read/write

Write mode:  [data|data|data|.......]
              0         ^limit=cap
                        position (sau mỗi write)

Sau flip():  [data|data|data|.......]
              0         ^limit
              ^position=0

Read mode:   đọc từ 0 đến limit
```

- `put()`: ghi vào position, tăng position
- `flip()`: limit = position, position = 0 → chuyển sang read mode
- `get()`: đọc từ position, tăng position
- `compact()`: sau partial read, copy unread data về đầu, ready to write more
- `clear()`: position=0, limit=capacity → discard all data, ready to write

**Channel vs Stream:**
- Stream: unidirectional, byte-oriented, blocking
- Channel: bidirectional, buffer-oriented, can be non-blocking

---

### 1.3 Selector Pattern (NIO Server)

```java
// Selector loop — event-driven server
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false); // PHẢI set non-blocking trước khi register
server.bind(new InetSocketAddress(8080));
server.register(selector, SelectionKey.OP_ACCEPT); // interested in accept events

while (true) {
    int ready = selector.select(1000); // block tối đa 1 giây, trả số channels ready
    if (ready == 0) continue;

    Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
    while (keys.hasNext()) {
        SelectionKey key = keys.next();
        keys.remove(); // PHẢI remove để tránh re-process

        if (key.isAcceptable()) {
            // New client connected
            SocketChannel client = ((ServerSocketChannel) key.channel()).accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ); // now interested in read
        } else if (key.isReadable()) {
            // Data available from client
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buf = ByteBuffer.allocate(4096);
            int bytes = client.read(buf);
            if (bytes == -1) { key.cancel(); client.close(); } // client disconnected
            else { buf.flip(); processRequest(client, buf); }
        }
    }
}
```

---

### 1.4 NIO.2 — File Operations

**Path vs File:**
- `File` (old): OS-dependent separators, limited methods, poor error messages
- `Path + Files` (Java 7+): type-safe, rich API, better exceptions, supports symlinks

```java
Path path = Path.of("/home/user/data.txt");

// Check
Files.exists(path)
Files.isReadable(path)
Files.isDirectory(path)

// Read (small files)
String content = Files.readString(path, StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(path);

// Read (large files — streaming)
try (BufferedReader reader = Files.newBufferedReader(path)) {
    reader.lines().forEach(System.out::println);
}

// Write (atomic via temp file)
Path temp = Files.createTempFile("prefix", ".tmp");
Files.writeString(temp, content);
Files.move(temp, path, StandardCopyOption.ATOMIC_MOVE); // atomic rename

// Walk directory tree
Files.walk(Path.of("/data"), 3) // max depth 3
    .filter(Files::isRegularFile)
    .filter(p -> p.toString().endsWith(".json"))
    .forEach(System.out::println);
```

---

### 1.5 HTTP Client (Java 11+)

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();

// Synchronous
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .GET()
    .header("Authorization", "Bearer " + token)
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.statusCode()); // 200
System.out.println(response.body());

// Asynchronous
CompletableFuture<HttpResponse<String>> future = client.sendAsync(request,
    HttpResponse.BodyHandlers.ofString());
future.thenApply(HttpResponse::body)
    .thenAccept(body -> processResponse(body));
```

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: ByteBuffer position lỗi

```java
// ❌ Lỗi phổ biến: quên flip() trước khi read
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.put("Hello".getBytes());
// position = 5, limit = 1024

byte[] bytes = new byte[buf.remaining()]; // remaining = limit - position = 1019
buf.get(bytes); // reads 1019 bytes từ position (mostly zeros)

// ✅ Phải flip()
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.put("Hello".getBytes()); // position = 5
buf.flip();                   // limit = 5, position = 0
byte[] bytes = new byte[buf.remaining()]; // remaining = 5
buf.get(bytes); // reads exactly "Hello"
```

### Issue 2: Channel read loop không handle partial reads

```java
// ❌ Chỉ đọc một lần — có thể partial (network fragmentation)
buf.clear();
int n = channel.read(buf);
processData(buf); // có thể thiếu data

// ✅ Read loop cho đến khi buffer full hoặc -1
buf.clear();
while (buf.hasRemaining()) {
    int n = channel.read(buf);
    if (n == -1) break; // end of stream
    if (n == 0) break;  // non-blocking: no data available right now
}
buf.flip();
```

### Issue 3: Resource leak với channels

```java
// ❌ Nếu exception → channel không được close
SocketChannel channel = SocketChannel.open(address);
doSomething(channel);
channel.close();

// ✅ Try-with-resources
try (SocketChannel channel = SocketChannel.open(address)) {
    doSomething(channel);
} // auto-closes even on exception
```

---

## 3. Code mẫu — Mini HTTP Server

```java
// http/HttpRequest.java
public record HttpRequest(
    String method,
    String path,
    Map<String, String> headers,
    String body
) {
    public String toRawHttp() {
        StringBuilder sb = new StringBuilder();
        sb.append(method).append(" ").append(path).append(" HTTP/1.1\r\n");
        headers.forEach((k, v) -> sb.append(k).append(": ").append(v).append("\r\n"));
        sb.append("\r\n");
        if (body != null && !body.isEmpty()) sb.append(body);
        return sb.toString();
    }
}

// http/HttpResponse.java
public record HttpResponse(int status, Map<String, String> headers, String body) {
    public static HttpResponse ok(String body) {
        return new HttpResponse(200, Map.of("Content-Type", "application/json"), body);
    }
    public static HttpResponse notFound(String message) {
        return new HttpResponse(404, Map.of("Content-Type", "application/json"),
            "{\"error\":\"" + message + "\"}");
    }
    public static HttpResponse badRequest(String message) {
        return new HttpResponse(400, Map.of("Content-Type", "application/json"),
            "{\"error\":\"" + message + "\"}");
    }

    public String toRawHttp() {
        String statusText = switch (status) {
            case 200 -> "OK"; case 201 -> "Created"; case 400 -> "Bad Request";
            case 404 -> "Not Found"; case 500 -> "Internal Server Error";
            default -> "Unknown";
        };
        StringBuilder sb = new StringBuilder("HTTP/1.1 ").append(status).append(" ").append(statusText).append("\r\n");
        headers.forEach((k, v) -> sb.append(k).append(": ").append(v).append("\r\n"));
        if (body != null) {
            sb.append("Content-Length: ").append(body.getBytes().length).append("\r\n");
        }
        sb.append("\r\n");
        if (body != null) sb.append(body);
        return sb.toString();
    }
}
```

```java
// http/RequestParser.java
public class RequestParser {

    public static HttpRequest parse(String raw) {
        // Split headers from body
        int headerEnd = raw.indexOf("\r\n\r\n");
        String headerSection = headerEnd >= 0 ? raw.substring(0, headerEnd) : raw;
        String body = headerEnd >= 0 ? raw.substring(headerEnd + 4) : "";

        String[] lines = headerSection.split("\r\n");
        if (lines.length == 0) throw new IllegalArgumentException("Empty request");

        // Parse request line: "GET /path HTTP/1.1"
        String[] requestLine = lines[0].split(" ");
        if (requestLine.length < 2) throw new IllegalArgumentException("Invalid request line");
        String method = requestLine[0];
        String path = requestLine[1];

        // Parse headers
        Map<String, String> headers = new LinkedHashMap<>();
        for (int i = 1; i < lines.length; i++) {
            int colon = lines[i].indexOf(": ");
            if (colon > 0) {
                headers.put(lines[i].substring(0, colon), lines[i].substring(colon + 2));
            }
        }

        return new HttpRequest(method, path, headers, body.isEmpty() ? null : body);
    }
}
```

```java
// server/HttpServer.java
import java.nio.*;
import java.nio.channels.*;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;

public class HttpServer {
    private final int port;
    private final Router router;
    private volatile boolean running = true;

    public HttpServer(int port, Router router) {
        this.port = port;
        this.router = router;
    }

    public void start() throws Exception {
        Selector selector = Selector.open();
        ServerSocketChannel server = ServerSocketChannel.open();
        server.configureBlocking(false);
        server.bind(new InetSocketAddress(port));
        server.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("Server started on port " + port);

        while (running) {
            selector.select(500);
            var iter = selector.selectedKeys().iterator();

            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove(); // must remove to prevent reprocessing

                try {
                    if (key.isAcceptable()) handleAccept(selector, key);
                    else if (key.isReadable()) handleRead(key);
                } catch (Exception e) {
                    System.err.println("Error handling key: " + e.getMessage());
                    key.cancel();
                    key.channel().close();
                }
            }
        }
    }

    private void handleAccept(Selector selector, SelectionKey key) throws Exception {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();
        if (client != null) {
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        }
    }

    private void handleRead(SelectionKey key) throws Exception {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buf = ByteBuffer.allocate(8192);
        int bytes = client.read(buf);

        if (bytes == -1) {
            key.cancel();
            client.close();
            return;
        }

        buf.flip();
        String rawRequest = StandardCharsets.UTF_8.decode(buf).toString();
        HttpRequest request = RequestParser.parse(rawRequest);
        HttpResponse response = router.route(request);

        // Write response
        ByteBuffer responseBuf = ByteBuffer.wrap(
            response.toRawHttp().getBytes(StandardCharsets.UTF_8)
        );
        while (responseBuf.hasRemaining()) client.write(responseBuf);
        client.close(); // HTTP/1.0: close after each response
    }

    public void stop() { running = false; }
}
```

```java
// client/RetryHttpClient.java
public class RetryHttpClient {
    private final HttpClient client;
    private final int maxRetries;
    private final long initialBackoffMs;

    public RetryHttpClient(int maxRetries, long initialBackoffMs) {
        this.client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
        this.maxRetries = maxRetries;
        this.initialBackoffMs = initialBackoffMs;
    }

    public HttpResponse<String> get(String url) throws Exception {
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .GET()
                    .timeout(Duration.ofSeconds(10))
                    .build();

                HttpResponse<String> response = client.send(request,
                    HttpResponse.BodyHandlers.ofString());

                if (response.statusCode() < 500) return response; // success or client error

                // 5xx → retry
                System.err.printf("Attempt %d/%d: server error %d for %s%n",
                    attempt + 1, maxRetries + 1, response.statusCode(), url);

            } catch (IOException | InterruptedException e) {
                System.err.printf("Attempt %d/%d: %s%n", attempt + 1, maxRetries + 1, e.getMessage());
            }

            if (attempt < maxRetries) {
                long backoff = initialBackoffMs * (1L << attempt); // 100, 200, 400ms
                System.out.printf("Waiting %dms before retry...%n", backoff);
                Thread.sleep(backoff);
            }
        }
        throw new RuntimeException("Max retries exceeded for: " + url);
    }
}
```

---

## 4. Lab Steps

1. **ByteBuffer lifecycle:** Tạo buffer, put 5 bytes, KHÔNG flip, read → observe garbage output. Thêm flip() → fix.
2. **NIO Server:** Start HttpServer port 8080 → `curl http://localhost:8080/api/items` → thấy JSON response
3. **Partial read:** Simulate fragmented data (send data in 2 ByteBuffer.write calls) → observe server handles it
4. **Retry client:** Gọi server đang down → observe exponential backoff logs → start server → request succeed
5. **File API:** Atomic write: write to temp, move to final path → tốt hơn trực tiếp write tới file (no partial write)
6. **Virtual threads + HTTP:** Thay NIO Selector bằng blocking server với virtual threads, benchmark throughput

---

## 5. Checklist tự kiểm tra

- [ ] ByteBuffer: luôn `flip()` sau write và trước read — đây là bug source #1
- [ ] NIO Selector: `keys.remove()` sau mỗi key processing — tránh reprocess
- [ ] Channel: `configureBlocking(false)` trước khi register với Selector
- [ ] `-1` từ `channel.read()`: client disconnect → cancel key, close channel
- [ ] Exponential backoff: `delay = initialDelay × 2^attempt` — tránh thundering herd
- [ ] `Files.move(temp, target, ATOMIC_MOVE)`: bảo đảm readers luôn thấy file complete hoặc file cũ
