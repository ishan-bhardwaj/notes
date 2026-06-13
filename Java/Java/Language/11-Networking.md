# Networking

## Connecting to a Server

### Sockets

```java
try (var s = new Socket("time-a.nist.gov", 13);
     var in = new Scanner(s.getInputStream())) {
    while (in.hasNextLine()) IO.println(in.nextLine());
}
```

- `Socket(host, port)` — opens TCP connection; throws `UnknownHostException` (subclass of `IOException`) if host unreachable
- `s.getInputStream()` / `s.getOutputStream()` — byte streams for reading/writing
- Both `read` and `write` block until data is available or connection fails
- Never use `Socket` for UDP — use `DatagramSocket` for that

### Socket Timeouts

```java
var s = new Socket();
s.connect(new InetSocketAddress(host, port), 10000); // 10s connection timeout
s.setSoTimeout(5000);  // 5s read timeout
```

- `Socket(host, port)` constructor can block indefinitely — prefer unconnected socket + `connect(address, timeout)`
- `setSoTimeout(ms)` — subsequent reads throw `SocketTimeoutException` if no data arrives in time
- No timeout exists for write operations
- `isConnected()`, `isClosed()` — query socket state

### Internet Addresses

```java
InetAddress addr = InetAddress.getByName("time-a.nist.gov");
InetAddress[] all = InetAddress.getAllByName("google.com"); // for load-balanced hosts
InetAddress local = InetAddress.getLocalHost();
byte[] bytes = addr.getAddress();
String ip = addr.getHostAddress();  // "129.6.15.28"
```

---

## Implementing Servers

### Server Sockets

```java
try (var s = new ServerSocket(8189);
     Socket incoming = s.accept()) {  // blocks until client connects
    var in = new Scanner(incoming.getInputStream());
    var out = new PrintWriter(incoming.getOutputStream(), true /* autoFlush */);
    out.println("Hello!");
    // echo loop...
    incoming.close();
}
```

- `ServerSocket(port)` — starts listening on given port
- `s.accept()` — blocks until a client connects; returns a `Socket` for that connection
- Each connection is a new `Socket` — independent from the `ServerSocket`

### Serving Multiple Clients

```java
ExecutorService service = Executors.newVirtualThreadPerTaskExecutor();
while (true) {
    Socket incoming = s.accept();
    service.submit(() -> serve(incoming));
}
```

- Use virtual threads (Java 21+) — one per connection; very lightweight
- `serve(incoming)` method handles one client; when it returns, the connection closes

### Half-Close

```java
try (var socket = new Socket(host, port)) {
    var in = new Scanner(socket.getInputStream());
    var writer = new PrintWriter(socket.getOutputStream());
    writer.print(requestData);
    writer.flush();
    socket.shutdownOutput();  // signal end of request; keep input open
    while (in.hasNextLine()) IO.println(in.nextLine());
}
```

- `shutdownOutput()` — send TCP FIN; server sees EOF on its input but can still write response
- Useful for one-shot protocols (HTTP-like): client sends everything, then reads everything
- `shutdownInput()` / `isInputShutdown()` / `isOutputShutdown()` — check/control each direction

### Interruptible Sockets

- Platform thread + regular `Socket`: blocking I/O is NOT interruptible via `Thread.interrupt()`
- __Virtual threads__: socket operations ARE interruptible — preferred approach in Java 21+
- Platform thread + `SocketChannel` (NIO): also interruptible

```java
SocketChannel channel = SocketChannel.open(new InetSocketAddress(host, port));
var in = new Scanner(channel);
OutputStream out = Channels.newOutputStream(channel);
```

- `SocketChannel` — interrupt causes `ClosedByInterruptException`; channel is closed
- Use NIO channels when you need interruptibility with platform threads

### Secure Sockets (TLS)

```java
SocketFactory factory = SSLSocketFactory.getDefault();
Socket s = factory.createSocket(host, port);

ServerSocketFactory factory = SSLServerSocketFactory.getDefault();
ServerSocket s = factory.createServerSocket(port);
```

- Configure via system properties: `-Djavax.net.ssl.keyStore=server.jks -Djavax.net.ssl.keyStorePassword=secret`
- Client needs trust store: `-Djavax.net.ssl.trustStore=client.jks -Djavax.net.ssl.trustStorePassword=secret`
- Debug: `-Djavax.net.debug=ssl`
- Generate certificates: `keytool -genkeypair ... -keystore server.jks`

---

## Getting Web Data

### URLs and URIs

- __URI__ — purely syntactic; parsed but not opened; no `openStream`
- __URL__ — can open a connection; only works with known schemes (`http`, `https`, `ftp`, `file`, `jar`)
- Construct `URL` from `URI` to avoid deprecated URL constructors:

```java
var url = new URI(urlString).toURL();
InputStream in = url.openStream();  // simplest way to read content
```

URI components: `scheme`, `schemeSpecificPart`, `authority`, `userInfo`, `host`, `port`, `path`, `query`, `fragment`

```java
URI base = new URI("https://docs.mycompany.com/api");
URI relative = new URI("../../java/net/Socket.html#Socket()");
URI combined = base.resolve(relative);
URI rel = base.relativize(combined);
```

### `URLConnection`

Steps (must follow this order):

1. `URLConnection connection = url.openConnection()`
2. Set request properties: `setDoOutput(true)`, `setRequestProperty(name, value)`, `setConnectTimeout`, `setReadTimeout`
3. `connection.connect()`
4. Read response headers: `getHeaderFields()` → `Map<String, List<String>>`, or convenience methods
5. `connection.getInputStream()` / `connection.getOutputStream()`

```java
URLConnection connection = url.openConnection();
connection.setRequestProperty("Authorization", "Basic " + Base64.getEncoder().encodeToString((user + ":" + password).getBytes()));
connection.connect();
Map<String, List<String>> headers = connection.getHeaderFields();
String type = connection.getContentType();
long modified = connection.getLastModified();
try (var in = new Scanner(connection.getInputStream(), "UTF-8")) { ... }
```

Convenience header methods: `getContentType()`, `getContentLength()`, `getContentEncoding()`, `getDate()`, `getExpiration()`, `getLastModified()`

### Posting Form Data

```java
URL url = new URI("https://host/path").toURL();
var connection = (HttpURLConnection) url.openConnection();
connection.setDoOutput(true);
try (var out = new PrintWriter(connection.getOutputStream())) {
    out.print(URLEncoder.encode(name, StandardCharsets.UTF_8) + "=" +
              URLEncoder.encode(value, StandardCharsets.UTF_8));
}
// switch to reading triggers the actual request
String encoding = connection.getContentEncoding();
try (var in = new Scanner(connection.getInputStream(), encoding != null ? encoding : "UTF-8")) { ... }
```

URL encoding rules: A–Z, a–z, 0–9, `. - ~ _` unchanged; space → `+`; all other chars → `%XX` (UTF-8 hex)

`HttpURLConnection` additional methods:
- `getResponseCode()` — HTTP status code
- `getErrorStream()` — read error body when `getInputStream()` throws
- `setInstanceFollowRedirects(false)` — manual redirect handling
- Check for `HTTP_MOVED_PERM`, `HTTP_MOVED_TEMP`, `HTTP_SEE_OTHER` and follow `Location` header

```java
CookieHandler.setDefault(new CookieManager(null, CookiePolicy.ACCEPT_ALL)); // handle cookies in redirects
```

---

## The HTTP Client (`java.net.http`)

### Building a Client

```java
HttpClient client = HttpClient.newHttpClient();  // defaults

HttpClient client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.ALWAYS)
    .executor(Executors.newCachedThreadPool())
    .build();
```

- `HttpClient` implements `AutoCloseable` — use in try-with-resources; `close()` waits for pending requests

### Building Requests

```java
HttpRequest request = HttpRequest.newBuilder()
    .uri(new URI(urlString))
    .header("Content-Type", "application/json")
    .GET()
    .build();

HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(new URI(urlString))
    .POST(HttpRequest.BodyPublishers.ofString(jsonString))
    .build();
```

Body publishers: `ofString(s)`, `ofByteArray(bytes)`, `ofFile(path)`, `concat(publishers...)`, `noBody()`

Modify existing request (Java 16+):

```java
HttpRequest modified = HttpRequest.newBuilder(request, (name, value) -> !name.equalsIgnoreCase("Content-Type"))
    .header("Content-Type", "application/xml")
    .build();
```

### Sending and Receiving

```java
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
String body = response.body();
int status = response.statusCode();
HttpHeaders headers = response.headers();
Optional<String> lastModified = headers.firstValue("Last-Modified");
Map<String, List<String>> headerMap = headers.map();
```

Body handlers: `ofString()`, `ofByteArray()`, `ofInputStream()`, `ofFile(path)`, `ofFileDownload(dir)`, `discarding()`

### Asynchronous Requests

```java
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenAccept(response -> processResponse(response.body()));
```

Logging: set system property `jdk.httpclient.HttpClient.log=headers,requests,content` and set logger level `jdk.httpclient.HttpClient.level=INFO`

---

## The Simple HTTP Server (JDK)

### Command-Line Tool

```bash
jwebserver                   # serves current directory on port 8000
jwebserver -d /tmp -p 8189  # custom directory and port
jwebserver -o verbose        # verbose logging
```

### Programmatic API

```java
import com.sun.net.httpserver.*;

HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
server.createContext("/", HttpHandlers.of(301, Headers.of("Location", "https://example.com"), ""));
server.createContext("/echo", exchange -> {
    String method = exchange.getRequestMethod();
    URI uri = exchange.getRequestURI();
    Headers reqHeaders = exchange.getRequestHeaders();
    String body = new String(exchange.getRequestBody().readAllBytes());
    
    byte[] responseBytes = "Hello".getBytes();
    exchange.getResponseHeaders().add("Content-Type", "text/plain");
    exchange.sendResponseHeaders(200, responseBytes.length);
    exchange.getResponseBody().write(responseBytes);
    exchange.getResponseBody().close();
});
server.start();
```

- `HttpHandlers.of(status, headers, body)` — simple fixed-response handler
- `HttpHandlers.handleOrElse(predicate, handlerA, handlerB)` — conditional dispatch
- `SimpleFileServer.createFileServer(address, root, outputLevel)` — static file handler

### Filters

```java
Filter logFilter = Filter.beforeHandler("logger", exchange -> {
    System.out.println(exchange.getRequestMethod() + " " + exchange.getRequestURI());
});
context.getFilters().add(logFilter);
```

- `Filter.beforeHandler(desc, consumer)` — runs before handler
- `Filter.afterHandler(desc, consumer)` — runs after handler
- `Filter.adaptRequest(desc, operator)` — modify request before handler
- Custom filter: extend `Filter`, implement `doFilter(exchange, chain)`; call `chain.doFilter(exchange)` to proceed

---

## Sending E-Mail (Jakarta Mail)

```java
Session mailSession = Session.getDefaultInstance(props);
var message = new MimeMessage(mailSession);
message.setFrom(new InternetAddress(from));
message.addRecipient(RecipientType.TO, new InternetAddress(to));
message.setSubject(subject);
message.setText(body);

Transport tr = mailSession.getTransport();
tr.connect(null, password);  // password is app-specific, not account password
tr.sendMessage(message, message.getAllRecipients());
tr.close();
```

Gmail properties:

```properties
mail.transport.protocol=smtps
mail.smtps.auth=true
mail.smtps.host=smtp.gmail.com
mail.smtps.user=accountname@gmail.com
```

- Use app-specific password, not account password (most providers require this)
- Debug: `mailSession.setDebug(true)`
- Requires Jakarta Mail JARs: `jakarta.mail-api`, `angus-mail`, `angus-activation`