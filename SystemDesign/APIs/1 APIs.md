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
