# Socket 

- A two-way communication channel between processes on different networked devices.
- Sockets are the lowest-level network abstraction used by all higher-level APIs (REST, GraphQL, gRPC).
- **Socket Address** - $(IP + Port + Protocol)$ uniquely identifies endpoints.
- **Connection** - Requires two socket endpoints (one per process).
```
Process A: IP1:Port1 ↔ Process X: IP2:Port2
```

- **Port Categories** -
  - Well-known ports (`1–1024`) - Standardized for common services.
    - HTTP - `80`
    - HTTPS - `443`
    - SMTP - `587`
    - PostgreSQL - `5432`

- **Multiple Concurrent Connections** -
  - A single machine can maintain multiple simultaneous socket connections to different services.
  - Each connection is uniquely identified by the 4-tuple: `(srcIP, srcPort, dstIP, dstPort)`.

> [!TIP]
> Transport Protocols - TCP or UDP determine data delivery guarantees.

## Categories of Network Sockets

- **Stream Sockets (TCP/SCTP)** -
  - Characteristics -
    - Connection-oriented - Establishes reliable end-to-end channel. 
    - Reliable - Automatic retransmission of lost packets.
    - Ordered - Guarantees delivery sequence.
  - TCP Connection States -
    - `CLOSED` → `LISTEN` (Server)
    - `SYN` → `SYN-ACK` → `ESTABLISHED` (3-way handshake)
    - `FIN` → `FIN-ACK` → `TIME_WAIT` → `CLOSED` (graceful close)
  - Features -
    - Sequence numbers track bytes sent/received.
    - ACKs confirm delivery.
    - Bidirectional communication until `FIN` closes the stream.
  
- **Datagram Sockets (UDP)** -
  - Characteristics -
    - Connectionless - No handshake — just sends data.
    - Unreliable - No guaranteed delivery or ordering.
    - Fire-and-forget - Best-effort delivery.
  - Use cases - Low-latency apps where occasional loss is acceptable (video streaming, gaming).
  - Limitations - No built-in recovery — application must handle if needed.

## Role of Sockets in API Development

- **Client Socket** - Initiates connection to server.
- **Server Socket** - Listens passively for incoming connections.
- **Client-Server Flow** -
  -  Server - `socket()` → `bind()` → `listen()`
  - Client - `socket()` → `connect()`
  - Server - `accept()` → Creates new socket for client
  - Client ↔ Server - `send()`/`recv()`
  - Both - `close()`

- **Server Socket** -
  - Lifecycle -
    - `socket()` — Create socket descriptor.
    - `bind()` — Associate with local IP:Port.
    - `listen()` — Enter passive mode, create connection queue.
    - `accept()` — Dequeue client, create dedicated connection socket.
    - Handle requests in separate threads/processes.
  - Scalability - Queue manages pending connections (FIFO).

- **Client Socket** -
  - Lifecycle -
    - `socket()` — Create socket.
    - `connect()` — Connect to known server (`IP:Port`).
    - `send()` / `recv()` — Exchange data.
    - `close()` — Terminate connection.
  - Requirement - Must know server socket address beforehand.

## Network Socket Interface (Common API Calls)

| Function    | Purpose                                                 |
| ----------- | ------------------------------------------------------- |
| `socket()`  | Creates socket descriptor (family, type, protocol).     |
| `bind()`    | Binds socket to local IP:Port.                          |
| `listen()`  | Server: Sets passive mode + connection queue size.      |
| `connect()` | Client: Establishes connection to server.               |
| `accept()`  | Server: Accepts pending connection, returns new socket. |
| `send()`    | Sends data from buffer to peer.                         |
| `recv()`    | Receives data into buffer from peer.                    |
| `close()`   | Closes connection, frees resources.                     |

## Real-World Example - MEVN Stack Network Statistics

```
Proto  Local Address       Foreign Address     State
TCP    0.0.0.0:80         0.0.0.0:*           LISTEN    (Server socket)
TCP    127.0.0.1:5432     127.0.0.1:54321     ESTABLISHED (DB connection)
TCP    104.18.2.119:443   10.2.y.z:54043      ESTABLISHED (HTTPS client)
TCP    *:3000             *:3000              TIME_WAIT (Node.js server)
```

- **Observations** -
  - `0.0.0.0` = Listen on all interfaces.
  - `LISTEN` = Server sockets waiting for connections.
  - `ESTABLISHED` = Active client-server communication.
  - `TIME_WAIT` = Graceful close pending.

## Web Sockets

- WebSockets provide full-duplex, bidirectional communication over a single TCP connection, addressing HTTP's limitations for real-time applications like chat and gaming.

- **HTTP Limitations** -
  - HTTP's request-response model suits batch tasks but struggles with real-time bidirectional needs.
  - Techniques like short polling waste resources with frequent requests, long polling ties up connections, and streaming remains half-duplex.

## WebSocket Mechanics

- Connections begin with an HTTP `GET` request including - `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key` (base64-encoded 16-byte nonce), and `Sec-WebSocket-Version: 13`. 
- Servers respond with HTTP 101 Switching Protocols, echoing `Sec-WebSocket-Accept` (SHA-1 hash of key + GUID "258EAFA5-E914-47DA-95CA-C5AB0DC85B11", base64-encoded).

- Data transfers via frames with opcodes: `0x1` (text), `0x2` (binary), `0x8` (ping), `0x9` (pong), `0xA` (close). 
- Control frames (`≤125 bytes`) handle state; clients mask payloads for security.

- Key Advantages -
  - Bidirectional, low-latency data exchange (2-10 byte headers).
  - Persistent connection reuses TCP, bypassing firewalls on ports `80`/`443`.
​
- Scaling Challenges -
Horizontal scaling requires sticky sessions or pub/sub since connections are stateful; load balancers can't easily route post-upgrade. Vertical scaling hits limits like file descriptors.
