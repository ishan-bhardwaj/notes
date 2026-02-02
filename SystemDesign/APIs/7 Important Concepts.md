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

