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