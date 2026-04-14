# API

- A software intermediary that allows two programs to communicate and exchange data/services.
- Abstraction - The client doesn’t need knowledge of the server’s internal workings — only the interface contract.

- Types of APIs -

| Type           | Access/Auth                    | Typical Users | Examples                    |
| -------------- | ------------------------------ | ------------- | --------------------------- |
| Public APIs    | Open with API keys             | B2C           | Google Maps, Weather APIs   |
| Private APIs   | Internal, no public access     | B2B, B2C, B2E | Internal APIs               |
| Partner APIs   | Controlled via tokens/licenses | B2B, B2C      | Amazon Partner APIs         |
| Composite APIs | Depends on connected APIs      | B2B, B2C, B2E | Stripe, PayPal Payment APIs |

> [!NOTE]
> Composite APIs - Combine multiple calls into one unified request — reduces latency and request overhead.

## API Requirements

| Type                        | Focus                                                 | Example                                                                              |
| --------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Functional Requirements     | Define what the API must do (features and behaviors). | Posting comments to a video in a streaming app.                                      |
| Non-Functional Requirements | Define how well the API performs — quality metrics.   | Response latency, scalability, reliability, availability, consistency, and security. |

## Characteristics of a Good API (Design)

| Characteristic                          | Explanation / Importance                                                                   |
| --------------------------------------- | ------------------------------------------------------------------------------------------ |
| Specification–Implementation Separation | Keep design independent from internal structure for flexibility and iterative improvement. |
| Concurrency                             | Handle multiple concurrent requests efficiently.                                           |
| Dynamic Rate Limiting                   | Prevent overload through adaptive throttling.                                              |
| Security                                | Implement proper auth, encryption, and data access controls.                               |
| Error Handling                          | Return clear, consistent error messages; reduce debugging overhead.                        |
| Architectural Style                     | Choose style (REST, GraphQL, gRPC, etc.) suited to use case.                               |
| Minimal but Comprehensive               | Compact yet covering all core operations effectively.                                      |
| Stateless or State-Bearing              | Favor idempotent operations for reliability.                                               |
| User Adoption                           | Aim for clear docs and usability—critical for community uptake.                            |
| Fault Tolerance                         | Continue functioning despite partial failures.                                             |
| Performance Measurement                 | Enable monitoring and early detection systems for reliability.                             |

## API Gateway

- Central entry point for all client requests in a microservices environment.
- Microservices benefit - Enables clients to interact via a unified endpoint instead of many individual service calls.

- **Roles** -
  - Routes requests to appropriate services.
  - Aggregates multiple service responses into one.
  - Performs security, authentication, and rate limiting.
  - Provides analytics and traffic monitoring.
  - Acts as a load balancer, aiding in system stability.

## API Endpoints

- Specific URLs that expose API operations/resources.
- **Structure** - `BASE_URL` + `endpoint_name`
- **Function** - Identify the location and type of operation (e.g., `/answers`, `/users`).

## API Performance Measures

- **Service Level Indicators (SLIs)** -
  - Quantitative metrics that measure specific aspects of service quality.
  - Common measures -

  | Indicator         | Definition / Meaning                                            |
  | ----------------- | --------------------------------------------------------------- |
  | Request latency   | Time taken to return a response.                                |
  | Error rate        | Ratio of failed requests to total requests.                     |
  | System throughput | Number of tasks or requests processed per unit time.            |
  | Availability      | Fraction of time the service is usable/operational.             |
  | Durability        | Probability that data remains intact and retrievable over time. |

- **Service Level Objectives (SLOs)** -
  - Target thresholds or ranges for SLIs, defining acceptable performance.
  - Purpose - Aligns user expectations and system design goals—avoids mismatch between perception and delivery.
  - Formula - $Bound_lower <= SLI <= Bound_upper$
  - Example -
    - “95% of API responses must be < 500 ms.”
    - “Availability should be at least 98%.”
  
- **Service Level Agreements (SLAs)** -
  - A contractual agreement between provider and consumer detailing service quality commitments.
  - Purpose - Protect customer interest and ensure trust by making service quality explicit and enforceable.
  - Includes -
    - Defined metrics (uptime, latency, throughput, error rate).
    - Measurement methods and frequency.
    - Consequences/remedies if targets aren’t met (e.g., compensation, service credits).

## Layered Architecture

- Each layer performs a specific function and communicates only with adjacent layers.
- The Nth layer provides services to the layer above and receives services from the layer below.
- Each layer defines -
  - Protocols for message order, structure, error handling, and delivery.
  - An interface that specifies available services to higher layers.

### OSI Model (7 Layers)

- **Application** – 
  - Interacts directly with user applications (e.g., HTTP, SMTP).

- **Presentation** – Data formatting, encryption/decryption.
  - Ensures data format compatibility between sender and receiver.
  - Handles encryption, decryption, and compression.

- **Session** – Manages sessions, authentication, reconnection.
  - Manages creation, maintenance, and termination of sessions.
  - Responsible for authentication and reconnection.

- **Transport** – Reliable data delivery (TCP).
  - Provides reliable data delivery, sequencing, and error checking.
  - Examples: TCP, UDP.

- **Network** – IP addressing and routing.
  - Handles routing, forwarding, and IP addressing.

- **Data Link** – Frame transmission, local addressing.
  - Manages local network communication; frames and MAC addressing.

- **Physical** – Transmission of raw bits over physical medium.
  - Transmits raw bits over the medium (Ethernet, DSL, fiber, etc.).

> [!NOTE]
> OSI is used as a conceptual reference, not a practical implementation.

> [!NOTE]
> The session and presentation layers were often ambiguous — their roles were absorbed into the application layer in real systems.

> [!TIP]
> Routers only deal with layers 1–3 (Physical, Data Link, Network).

### TCP/IP Model (Internet Model)

- Practical implementation that powers the Internet.
- Layers -
  - **Application Layer** -
    - Protocols for user-level data handling (HTTP, SMTP, FTP, etc.).
  - **Transport Layer** -
    - Ensures communication between applications (e.g., TCP, UDP) using port numbers.
  - **Internet Layer** -
    - Core = IP (Internet Protocol) for addressing and routing packets.
  - **Network Access Layer** -
    - Combines Physical + Data Link layers; includes Ethernet, Wi-Fi, etc.

## Throughput

- Process-to-process data rate across network hops.
- **Formula** - Data transferred per unit time (e.g., Mbps).
- **Constraint** - Limited by the slowest link or endpoint (e.g., IoT device at `1 Mbps` on a `2 Mbps` network = bottleneck at `1 Mbps`).

## Latency

- Time for a user-level message to travel from sender to receiver.
- $Latency = Transmission delay + Propagation delay + Queuing delay + Processing delay$

- **Transmission Delay** -
  - Time to push packet bits onto the transmission medium.
  - $Transmission delay = Packet Size / Bandwidth$
  - Tradeoffs -
    - Larger packets → higher delay.
    - Higher bandwidth → lower delay.
  - Optimization - Compress data to reduce packet size.

- **Propagation Delay** -
  -  Time for bits to travel from source to destination (speed of light limitation).
  - $Propagation Delay = Distance / Speed$
  - Examples -
    - Cross-continent - `150–250 ms`
    - Earth to Mars - `~7.4 minutes` (`134M km` at speed of light).
  - Implication - Geographic distance fundamentally limits latency — use CDNs or edge computing for mitigation.

- **Queuing Delay** -
  - Time packets wait in router queues before transmission.
  - Factors -
    - Packet arrival rate vs. link capacity.
    - Current network congestion.
    - Number of packets ahead in queue.
  - Analogy - Traffic lights — packets wait their turn.
  - Mitigatio - QoS policies, traffic shaping, congestion control.

- **Processing Delay** -
  - Time routers/end hosts spend processing packets (header parsing, routing lookups).
  - Factors - Hardware/software speed, packet complexity.
  - Critical for - High-frequency trading (milliseconds matter).
  - Optimization - Faster hardware, optimized routing tables, offloading to ASICs.

## Jitter

- Variation in packet latency between consecutive packets.
- Calculation - Mean of absolute differences in packet arrival times.
- Example - Ping times - `11.0`, `10.3`, `10.3`, `10.3`, `10.2 ms` → Jitter = `0.2 ms`.
- Impact -
  - Video/audio streaming → buffering required.
  - Real-time apps → quality degradation.
- Causes - Route changes, queue fluctuations, congestion.

## Latency-Bandwidth Product (LBP)

- Maximum amount of data the network can hold in transit.
- Formula - $LBP = Latency * Bandwidth$
- Example - $200 ms latency × 1 Mbps = 200 kbits in flight$
- Implication -
  - Send LBP worth of data before waiting for `ACK` to maximize throughput.
  - Underutilization occurs if you send less than LBP.
- RTT-Bandwidth Product - $LBP × 2$ (for round-trip).

> [!TIP]
> Application awareness guides API design -
>   - Elastic (Can tolerate delay) → Optimize throughput.
>   - Real-time (Delay-sensitive) → Minimize latency + jitter.

## API Architectural Styles

| Style      | Protocol               | Data Format               | Strengths                                         | Weaknesses                                                |
| ---------- | ---------------------- | ------------------------- | ------------------------------------------------- | --------------------------------------------------------- |
| REST       | HTTP/1.1               | JSON/XML                  | Simple, cacheable, widely supported               | Over/under-fetching data, multiple endpoints educative+1​ |
| GraphQL    | HTTP (single endpoint) | JSON                      | Flexible queries, no over-fetching, strong typing | Server complexity, caching challenges educative+1​        |
| gRPC (RPC) | HTTP/2                 | Protocol Buffers (binary) | High performance, streaming, multi-language       | Browser limitations, steeper learning curve educative+1​  |
| SOAP (RPC) | HTTP/SMTP              | XML                       | Strict standards, enterprise security             | Verbose, rigid, legacy overhead educative+1​              |

# API Architecture Styles

## REST 

- REST is an architectural style for web APIs defined by Roy Fielding in 2000 to enhance scalability, performance, and flexibility of distributed systems like the early World Wide Web.

- REST enforces six key constraints that guide API design, enabling independent evolution of components while maintaining system-wide properties -
  - **Client-Server** - Separates concerns between client (UI-focused) and server (data-focused) components, improving scalability through independent development.
  - **Stateless** - Each request contains all necessary information; servers retain no session state, boosting reliability, scalability, and visibility but requiring repetitive data transmission.
  - **Cacheable** - Responses are marked explicitly cacheable or non-cacheable, reducing latency via client-side caching at the cost of potential staleness.
  - **Uniform Interface** - Standardizes interactions via resource identification (URIs), representations (JSON/XML), self-descriptive messages (status codes), and HATEOAS (hypermedia links for discoverability).
  - **Layered System** - Supports intermediaries (proxies, load balancers) for security and load distribution, enhancing evolvability and reusability but adding latency.
  - **Code-on-Demand** (Optional) - Servers send executable code (scripts/applets) to extend client functionality, improving performance but reducing visibility and introducing security risks.
​
- REST APIs map CRUD operations directly to HTTP methods for predictable behavior -

| CRUD             | HTTP Method | Purpose                                         | Idempotent?      |
| ---------------- | ----------- | ----------------------------------------------- | ---------------- |
| Create           | POST        | Creates new resource (e.g., POST /users)        | No educative+1​  |
| Read             | GET         | Retrieves resource(s) (e.g., GET /users/123)    | Yes educative+1​ |
| Update (full)    | PUT         | Replaces entire resource (e.g., PUT /users/123) | Yes educative+1​ |
| Update (partial) | PATCH       | Modifies parts of resource                      | No educative+1​  |
| Delete           | DELETE      | Removes resource (e.g., DELETE /users/123)      | Yes educative+1​ |

## GraphQL 

- GraphQL serves as a query language for APIs, enabling precise data fetching from multiple endpoints in a single request, addressing key limitations of REST APIs. 
- Developed by Facebook in 2012 and open-sourced in 2015, it provides a flexible alternative for complex applications like social media platforms.
- **REST API Limitations** -
  - REST APIs often require multiple requests to gather data from various endpoints, such as separate calls for starships, planets, and films from SWAPI. 
  - They also suffer from overfetching (retrieving excess data beyond needs) and underfetching (lacking sufficient data, leading to additional requests and the `n+1` problem).

- **GraphQL Components** -
  - GraphQL systems consist of a server with a schema defining data types and relationships, plus resolver functions linking fields to backends. 
  - The client—such as web or mobile apps—sends queries to a single endpoint, receiving only requested data in JSON format matching the query shape.
​
- **Key Operations** -
  - Queries fetch specific nested data, like starship name, length, and cargo capacity alongside planet details, all in one request. 
  - Mutations handle data changes (insert, update, delete) akin to REST's POST/PUT/DELETE, such as adding a student record with name, rollNo, and degreeProgram.
​
- **Advantages Over REST** -
  - GraphQL eliminates multiple requests and over/underfetching by allowing exact data specification, reducing round trips and bandwidth.

- **Drawbacks** -
  - It complicates error handling (parsing responses instead of status codes), file uploads (requiring Base64, separate servers, or libraries), and caching across endpoints.

## gRPC 

- gRPC excels as a high-performance RPC framework for internal service-to-service communication, leveraging HTTP/2 multiplexing and Protocol Buffers for superior throughput compared to REST over HTTP/1.1, particularly in low-latency, high-throughput scenarios like microservices.
- Its binary serialization and streaming capabilities reduce payload sizes and enable efficient handling of multiple concurrent requests.

- **Origin** -
  - Google developed gRPC from the internal "Stubby" project starting in 2001, releasing it as open-source in 2015 and incubating it under CNCF in 2017 to overcome limitations in SOAP, JSON-RPC, and REST such as tight coupling and header overhead.

- **Core Communication Types** - 
  - gRPC supports four connection patterns via HTTP/2 subchannels: unary request-response, client streaming (multiple client messages, single response), server streaming (single request, multiple responses), and bidirectional streaming.

- **Key Components** -
  - Channels establish HTTP/2 connections to endpoints, spawning subchannels for load-balanced streams across backends. 
  - Pluggable elements include DNS resolvers for IP discovery, load balancers for backend selection, and proxies like Envoy for HTTP/1.1 compatibility.

- **Advantages Over REST** -
  - gRPC delivers 3x higher throughput than JSON/HTTP in asynchronous scenarios due to compact Protobuf payloads and HTTP/2 features like flow control, with automatic code generation from machine-readable contracts.
​
- **Limitations** -
Browser support requires proxies (lacking native HTTP/1.1 compatibility), binary formats hinder debugging, and POST-only requests prevent standard HTTP caching.

## REST vs GraphQL vs gRPC 

| Style   | Use When                                            | Avoid When                                                      |
| ------- | --------------------------------------------------- | --------------------------------------------------------------- |
| REST    | Few resources, CRUD ops, stateless, browser clients | Event-driven systems, big data analyticseducative​              |
| GraphQL | Multi-source aggregation, shared data varying views | Server-to-server communicationeducative​                        |
| gRPC    | Microservices, low latency, high throughput         | Unsupported languages, limited backends, browser-onlyeducative​ |

# API Versioning

- API Versioning enables controlled evolution of APIs while maintaining backward compatibility for existing clients. 
- Good designs minimize versioning needs through stable specs, but changes require strategies to avoid breaking applications used by millions.

- **Versioning Principles** -
  - Backward Compatibility - New versions must support old clients; servers handle prior version requests gracefully.
  - Low Frequency - Limit versions to reduce provider maintenance and client upgrade burden.
  - Skip Non-Breaking Changes - Add query params, new fields, or optional features without versioning.
  - Server Backward, Not Forward - Servers accommodate old clients; old servers reject new client format.

- **Version Compatibility Matrix** -

| Client \\ Server | Old     | New (BC) | New (Breaking) |
| --------------- | ------- | -------- | -------------- |
| Old             | ✅ Works | ✅ Works  | ❌ Fails        |
| New             | ✅ Works | ✅ Works  | ✅ Works        |

## When to Version

- **Version Required (Breaking Changes)** -
  - Structural changes - string → object (e.g., `mobileinfo: "iPhone"` → `{model: "iPhone", price: 999}`)
  - Entity renaming - `userID` → `userCredentials`
  - Functionality splits - monolithic → microservices
  - Performance changes affecting contracts

- **No Versioning (Safe Changes)** -
  - Add optional fields/query params.
  - Remove obsolete entities (after deprecation).
  - Backward-compatible data format evolution.

## Versioning Approaches

| Method      | Example                                 | Pros                 | Cons                                |
| ----------- | --------------------------------------- | -------------------- | ----------------------------------- |
| URL         | `/v1/users`, `/v2/users`                    | Simple, discoverable | URL pollution, high maintenance     |
| HTTP Header | Accept: `application/vnd.example.v2+json` | Clean URLs           | Client complexity, debugging harder |
| Hostname    | `api.example.com` → `v2.api.example.com`    | Full isolation       | DNS changes, major rewrites only    |

> [!TIP]
> Most Common - URI versioning (`/v1/`, `/v2/`) due to simplicity.

- **Lifecycle Management** -
  - Announce - Publish schedules, send emails to registered users.
  - Warn - Add deprecation headers (Sunset: 2026-01-01) in old versions.
  - Beta - Release to select developers for feedback.
  - Migration Window - Support old versions 6-18 months post-launch.
  - Graceful Retirement - Redirect or 410 after end-of-life.

### API Evolution vs Versioning

- Evolution = same version, backward compatible.
- Versioning = breaking changes, new version needed.

# Rate Limiting

- Controls API request volume to protect infrastructure from overload.
- Throttles excess requests instead of disconnecting clients.
- Prevents DoS attacks, bots, and "thundering herds" (mass simultaneous requests).

- **Core Responsibilities** - 
  - Consumption quota - Max API calls per time frame per app.
  - Spike arrest - Blocks sudden traffic surges.
  - Usage throttling - Slows heavy users during peak hours.
  - Traffic prioritization - Higher quotas for premium vs freemium.
​
- **How It Works -
  - Checks client request count vs limit in DB/cache.
  - Allows request if under limit, increments counter.
  - Rejects/throttles if limit exceeded.

- **Placement Options** -
  - Client-side - Easy but insecure (clients can bypass)
  - Server-side - API server handles limiting logic
  - Middleware - Isolated service (preferred for scalability)

- **HTTP Response Standards** -
  - 429 Too Many Requests status code
  - `retry-after header` - When to retry
  - `X-RateLimit-Limit` - Max requests allowed
  - `X-RateLimit-Remaining` - Requests left
  - `X-RateLimit-Reset` - Reset timestamp
  - `X-RateLimit-Resource` - Affected resource

- **Best Practices** -
  - Choose algorithm by traffic pattern (Token Bucket, Leaky Bucket, Fixed/Sliding Window).
  - Set thresholds based on user count/typical behavior.
  - Document limits clearly upfront.
  - Start low, scale quotas gradually.
  - Implement exponential backoff in SDKs.

# Client-Adapting APIs

- Servers tailor responses based on client device, location, or context (mobile vs desktop, region-specific features).
- **Problem** - One-size-fits-all APIs waste bandwidth, overload clients, or deliver irrelevant data.
- **Client-Side Flexibility** - Same response for all → client filters locally (Educative shows 4/8 images on mobile).
  - Simple server logic.
  - Wastes bandwidth on unused fields.

- **Server-Side Flexibility (3 approaches)** -
  - IAC (Netflix) - Device-specific endpoints (800+), middleware detects client → aggregates → tailors response.
  - Expandable Fields - `?expand=images` adds conditional data to single endpoint.
  - GraphQL - Client queries exact schema → single aggregated response from multiple services.

- **Advantages** - Better UX, efficient bandwidth/CPU, optimal processing location.
- **Drawbacks** - Complex server code, $$$ maintenance, over-engineering risk.
- **Best for chat apps** - Server-side (IAC) → handles device/version compatibility centrally.

# Data Fetching Patterns

- Techniques for real-time client-server communication (read-heavy apps like Twitter/Quora).
- **Short Polling** -
  - Client requests data at fixed intervals (<1min).
  - Server responds immediately (data or empty).
  - Problems - Needless requests, delays until next poll.

- **Long Polling (Hanging GET)** -
  - Client request stays open until server has data or timeout.
  - Better than short polling but still reconnection overhead.
  - Problems - Server manages many open connections, client reconnections.
​
- **WebSocket** -
  - Full-duplex persistent TCP connection (HTTP → WS upgrade).
  - Bidirectional, low latency, no polling.
  - Perfect for chat, games, notifications.

- **Comparison** -

| Approach  | Low Latency | Bandwidth | Full Duplex | Browser Support |
| --------- | ----------- | --------- | ----------- | --------------- |
| Short     | ❌           | ❌         | ❌           | ✅               |
| Long      | ✅           | ✅         | ❌           | ✅               |
| WebSocket | ✅           | ✅         | ✅           | ✅               |

# Event Driven Architecture

- Event-driven architecture protocols enable real-time pushes, addressing HTTP polling inefficiencies for scenarios like notifications and live updates.
- **Core Protocols** -
  - Webhooks - Unidirectional HTTP callbacks from server to server; subscribe via endpoint for events like Slack-Google Calendar sync; no retry guarantees.
  - SSE (Server-Sent Events) - Server-to-client text streaming over HTTP; auto-reconnects; ideal for feeds (e.g., Twitter updates); browser connection limits apply.
  - WebSub (Pub/Sub) - Hub-based distribution; publisher → hub → subscribers with verification; scales for multi-client syndication.
  - WebSocket - Bidirectional persistent connection; supports binary; for interactive apps like chat/gaming.

- **Key Comparisons** -
  - HTTP-based - All except WebSocket (upgrades HTTP); enables firewall compatibility.
  - Direction - Webhook/SSE unidirectional; WebSub unidirectional fan-out; WebSocket bidirectional.
  - Consumers - Webhook servers; SSE/WebSub clients/servers; WebSocket both.
​ - Binary data - No for SSE; yes for others.
  - Reliability - SSE auto-reconnect; WebSub verification; webhooks/WebSocket need custom retries.

- **Selection Guide** -
  - Server notifications - Webhooks (simple, direct).
  - Client text streams - SSE (lightweight).
  - Many-to-many - WebSub (decoupled).
​ - Interactive - WebSocket (full duplex).
