# Load Balancers

- A Load Balancer (LB) distributes incoming client requests across a pool of servers.
- Prevents overloading or crashing of servers.
- Acts as the first point of contact in a data center after the firewall.
- Not required for small traffic (hundreds/thousands of requests per sec), but essential for high traffic.
- Capabilities -
    - Scalability - Add/remove servers seamlessly; transparent to end users.
    - Availability - Hides server faults/failures; system stays available.
    - Performance - Forwards requests to less-loaded servers; improves response time and utilization.

> [!TIP]
> Placement of Load Balancers can be -
>   - Between clients ↔ web servers/application gateway.
>   - Between web servers ↔ application servers.
>   - Between application servers ↔ database servers.
>   - Can be used between any two multi-instance services.

- Services Offered by Load Balancers
    - Health Checking - Uses heartbeat protocols to monitor end-server health.
    - TLS Termination - Offloads TLS/SSL handling from servers.
    - Predictive Analytics - Identifies traffic patterns for optimization.
    - Reduced Human Intervention - Automation minimizes manual system admin tasks.
    - Service Discovery - Forwards requests to appropriate hosting servers via service registry.
    - Security - Mitigates DoS and other attacks at OSI layers 3, 4, and 7.

## Types of Load Balancers

### Global Server Load Balancing (GSLB)

- Distributes traffic across multiple geographical regions (different data centers).
- Ensures failover when a data center suffers power or network failure.
- Decisions are based on -
    - User’s geographic location
    - Number of hosting servers in each region
    - Health of data centers
    - Monitoring information from local LBs
- Can be installed on-premises or obtained via Load Balancing as a Service (LBaaS).
- Supports automatic zonal failover.

### Local Load Balancing

- Operates within a data center.
- Distributes requests among servers to maximize efficiency and resource utilization.
- Acts as a reverse proxy, using a Virtual IP (VIP) to accept client connections.

## GSLB with DNS

- DNS can return multiple IP addresses for a query.
- Uses round-robin technique: reorders IPs for each DNS response, so different users connect to different servers.
- Limitations of DNS Round-Robin -
    - ISPs with many customers cache one IP, leading to uneven load distribution.
    - Doesn’t detect server crashes - keeps sending traffic to failed servers until TTL expires.
    - Recovery from failures is slow due to DNS caching (especially with long TTL).
    - Packet size limit (512 bytes) restricts how many IPs can be included in responses.
    - Limited control -
        - Clients may choose arbitrarily from provided IPs.
        - Clients can’t always connect to the closest data center.
- Despite limitations, short TTLs are used to improve balancing.
- DNS GSLB is simple but limited; better alternatives are Application Delivery Controllers (ADCs) and cloud-based load balancing.

## Anycast in Global Load Balancing

- Anycast - Same IP address is assigned to multiple servers in different geographic locations.
- Routing automatically directs requests to the nearest server (based on routing tables).
- Advantages over traditional DNS GSLB -
    - Faster response times (clients connect to geographically nearest server).
    - No dependency on DNS caching - avoids stale/failed IP issues.
    - Better fault tolerance - if a server fails, routing naturally redirects to the next nearest server.
    - Works at the network layer, making it more reliable and transparent.

## Why Local Load Balancers Are Still Needed

- DNS/Global balancing cannot solve everything because -
    - Limited DNS packet size (512 bytes).
    - Clients may not pick optimal IPs.
    - No guarantee of connecting to the nearest/least-loaded server.
    - Slow recovery due to caching delays.
- Local Load Balancers -
    - Provide finer control inside the data center.
    - Handle real-time health checks and intelligent request routing.
    - Ensure requests are distributed fairly across active servers.

## Load Balancers Algorithms

- **Round-robin scheduling** - Each request is sent to servers sequentially in a cycle.
- **Weighted round-robin** - Servers with higher capacity are assigned greater weights, so they handle more requests.
- **Least connections** - Requests go to servers with the fewest active connections. Useful when -
    - Clients have long-lived requests.
    - Uneven load distribution occurs despite equal server capacity.
    - LB must track and maintain state of current connections.
- Least response time - Requests go to the server with the fastest response time (performance-sensitive systems).
- IP hash - Requests are assigned based on a hash of the client’s IP address. Ensures consistent server assignment for specific users.
- URL hash - Requests mapped using a hash of the URL, useful when certain services/paths are hosted on specific servers.

## Static vs Dynamic Algorithms

- Static Algorithms -
    - Do not consider real-time server state.
    - Decisions based only on pre-known configuration (e.g., round-robin).
    - Simple and lightweight - typically run on a single router or commodity machine.
    - Low complexity but less adaptive.
- Dynamic Algorithms -
    - Consider current or recent state of servers.
    - Require communication between load balancers to exchange state.
    - Add complexity and overhead but make smarter forwarding decisions.
    - Monitor server health and only send traffic to active servers.
    - Provide better results in practice.

## Stateful vs Stateless Load Balancing

- Stateful Load Balancing -
    - Maintains session state between clients and servers.
    - Keeps a mapping table of client - server.
    - Load balancers share session state with each other for consistent forwarding.
    - Provides reliability but increases complexity and reduces scalability.
- Stateless Load Balancing -
    - Keeps no global session state.
    - Uses consistent hashing (e.g., hash of client/session info) to map requests.
    - Lightweight and faster.
    - Less resilient if infrastructure changes (new/removed servers), as consistent hashing alone may not be enough.
    - May still keep local state internally, but not shared globally.
- Rule of thumb -
    - Stateful LB - higher reliability, but complex.
    - Stateless LB - faster, simpler, but less resilient to infrastructure changes.

## Types of Load Balancers (OSI Layers)

- Layer 4 Load Balancers (Transport-level) -
    - Operate at TCP/UDP level.
    - Ensure packets from one connection go to the same back-end server.
    - Some L4 LBs support TLS termination.
    - Faster (less inspection).
- Layer 7 Load Balancers (Application-level) -
    - Application-aware - make decisions based on HTTP headers, cookies, URLs, user IDs, etc.
    - Perform TLS termination, request routing, rate limiting, header rewriting.
    - Smarter but slower (more inspection).
- Trade-off -
    - L4 LBs - faster.
    - L7 LBs - more intelligent decisions.

## Load Balancer Hierarchy in Data Centers

- Large data centers use multi-tier LB architectures.
- **Tier-0 LB** -
    - DNS acts as the entry-level LB.
    - Provides initial distribution across data centers.
- **Tier-1 LB** -
    - ECMP (Equal Cost Multipath) routers.
    - Distribute incoming traffic across Tier-2 LBs.
    - Use IP-based or simple algorithms like round-robin.
    - Provide horizontal scalability to higher-tier LBs.
- **Tier-2 LB** -
    - Typically Layer 4 LBs.
    - Ensure that packets from one connection are consistently forwarded to the same Tier-3 LB.
    - Use techniques like consistent hashing.
    - Act as glue between Tier-1 and Tier-3.
    - Prevents errors in case of failures or dynamic scaling.
- **Tier-3 LB** - 
    - Layer 7 LBs in direct contact with application servers.
    - Perform TLS termination and application-level routing.
    - Monitor server health at HTTP level.
    - Reduce burden on servers by handling -
        - TCP congestion control
        - Path MTU discovery
        - Protocol translation between client and servers
- Summary of Tiers -
    - Tier-1 - distributes load across LBs.
    - Tier-2 - ensures smooth forwarding consistency.
    - Tier-3 - actual request distribution among servers.
