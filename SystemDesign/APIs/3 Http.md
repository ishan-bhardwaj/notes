## World Wide Web (WWWW)

- Hypertext-based system linking documents across millions of Internet machines.
- Commonly called web or w3.
- Client-Server Model -
  - Client requests resources.
  - Server provides requested resources via APIs.
- Focus protocols - HTTP (WWW), RPC, WebSockets — cover major client-server communication patterns.

- **Internet vs Web** -
  - Internet - The underlying network infrastructure.
  - Web - The hyperlinked content and services running on top of the Internet.

### Client-Server Model

- **Client** - Requests resources (e.g., browser querying DNS).
- **Server** - Delivers resources.
- **Example** - Browser → `google.com` → DNS → IP `102.16.5.110`

### Resource Identifiers (URI)

```
https://www.google.com/maps
```

- Protocol - `https`
- Domain - `www.google.com`
- Path - `/maps`

### Hypertext & Hypermedia

- **Hypertext** - Non-linear text with embedded links.
- **Hypermedia** - Hypertext + multimedia (images, video, interactive elements).

### Markup Language (HTML)

- Defines document structure, formatting, and links.
- Browsers parse HTML to render web pages.

## Web Protocols

- **Primary** - HTTP (Hypertext Transfer Protocol) — defines client-server request/response format.
- **Lower layers** - TCP/UDP + IP handle transport and routing.
- **Other protocols** - FTP, SMTP (also application layer, built on TCP/IP).

## How does the Web work?

### Client-Side Flow

- User clicks link → Browser extracts URL
- Browser queries DNS → Resolves domain to IP
- Browser establishes TCP connection to server IP
- Browser sends HTTPS request for resource
- Server responds with HTML + assets
- Browser fetches additional objects (images, CSS, JS)
- Browser renders complete page
- TCP connection closes

### Server-Side Flow

- Server accepts TCP connection from client
- Server receives HTTP request (URI + headers)
- Server retrieves resource (DB/cache)
- Server generates HTTP response
- Server sends response to client
- Server closes TCP connection

> [!TIP]
> Server uses 5-tuple to match responses to requests: `(srcIP, srcPort, protocol, dstIP, dstPort)`

## HTTP

- HTTP (Hypertext Transfer Protocol) is a stateless, application-layer protocol for distributed hypermedia systems.
- _Stateless_ - Each request independent - no session memory between requests.

- **HTTP Message Structure** -
  - Request format -
    ```
    METHOD Request-Target HTTP-Version
    Headers...
    Empty Line
    Body (optional)
    ```
    - Method - Action (GET, POST, PUT, etc)
    - Request-Target - URL/URI identifying resource
    - Headers - Metadata (key-value pairs)
    - Body - Application data

  - Response format -
    ```
    HTTP-Version Status-Code Status-Phrase
    Headers...
    Empty Line
    Body
    ```

- **Status Codes** -
| Range   | Meaning       | Examples                       |
| ------- | ------------- | ------------------------------ |
| 100-199 | Informational | 100 Continue                   |
| 200-299 | Success       | 200 OK, 201 Created            |
| 300-399 | Redirection   | 301 Moved Permanently          |
| 400-499 | Client Error  | 400 Bad Request, 404 Not Found |
| 500-599 | Server Error  | 500 Internal Server Error      |

- **HTTP Methods** -
| Method  | Purpose                 | Safe | Idempotent | Example                   |
| ------- | ----------------------- | ---- | ---------- | ------------------------- |
| GET     | Read resource           | ✅    | ✅          | curl https://example.com  |
| POST    | Create new resource     | ❌    | ❌          | Login, form submission    |
| PUT     | Replace entire resource | ❌    | ✅          | Full profile update       |
| DELETE  | Remove resource         | ❌    | ✅          | Delete user               |
| HEAD    | Headers only (fast)     | ✅    | ✅          | Check existence/timestamp |
| OPTIONS | Server capabilities     | ✅    | ✅          | CORS preflight            |
| TRACE   | Debug (echo request)    | ✅    | ✅          | Rarely used               |
| CONNECT | Tunnel (proxy)          | ❌    | ❌          | HTTPS via proxy           |

- **Essential Headers** -
| Header            | Direction | Purpose                       |
| ----------------- | --------- | ----------------------------- |
| Accept            | Request   | Content types client accepts  |
| Authorization     | Request   | Client credentials            |
| Content-Type      | Both      | Data format (JSON, form-data) |
| Content-Length    | Both      | Body size in bytes            |
| Cache-Control     | Both      | Caching directives            |
| Server            | Response  | Server software info          |
| If-Modified-Since | Request   | Conditional GET               |

> [!TIP]
> Always validate method idempotency/safety for retry logic in distributed systems.

## HTTP Versions

### HTTP/0.9 (1991)

- Simplest version with one-line GET requests only—no headers, status codes, or error handling.
- Responses limited to hypertext (HTML); connection terminates immediately after response.
- Limitations - No support for other methods, content types, caching, or concurrent connections; insecure without encryption.
​
### HTTP/1.0 (1996)

- Introduced headers for metadata, status codes (errors/redirection), and methods (HEAD, GET, POST).
- Supported diverse content (scripts, media) beyond hypertext; added conditional requests and encoding.
- Drawbacks - Non-persistent connections required TCP handshake per request; no mandatory Host header (hindered virtual hosting); limited caching via If-Modified-Since.
​

### HTTP/1.1 (1997)

- Enabled persistent connections (reuse TCP for multiple requests/responses, reducing handshake overhead).
- Added pipelining (send multiple requests before responses), mandatory Host header, virtual hosting, more methods (PUT, DELETE, TRACE, OPTIONS), chunked transfers, compression, Upgrade header, and advanced cache headers (If-Unmodified-Since, If-Match).
- Key benefits - Lower latency, better caching control, proxy support; most widely used legacy version.
​

### HTTP/2 (2015)

- Backward-compatible binary protocol over persistent TCP; introduced multiplexing (multiple interleaved requests/responses without blocking).
- Features - Server push (pre-send resources), header compression (HPACK reduces overhead), stream prioritization; responses arrive out-of-order with IDs mapping to requests.
- Benefits - Faster page loads, optimized bandwidth; based on Google's SPDY.
​

### HTTP/3 (2019)

- Shifts to QUIC over UDP (replaces TCP) for faster connection setup (combines transport/TLS handshakes), stream-level reliability, and congestion control.
- Eliminates head-of-line blocking via multiplexing; integrates TLS 1.3; resilient to network changes (e.g., mobile Wi-Fi to cellular).
- Benefits - 33% faster connections, lower latency, better mobile performance; up to 3x faster than HTTP/1.1.
