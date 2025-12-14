# Domain Name System (DNS)

- DNS (Domain Name System) is the Internet’s naming service - maps human-friendly domain names (e.g., `google.com`) to machine-readable IP addresses (e.g., `142.250.187.142`).
- The process is transparent to users: when a user types a URL, DNS resolves it automatically.
- Once the IP address is obtained, the browser forwards the request to the destination web server.

- Sequence of a DNS Lookup -
    - The user enters a domain name in the browser.
    - The browser queries the DNS infrastructure to resolve the domain name.
    - The DNS name server responds with the IP address.
    - The browser receives the IP address and connects to the web server.


## Key Concepts

### Name Servers

- DNS is not a single server - it is a large infrastructure with many servers worldwide.
- Servers that answer user queries are called Name Servers.
- Types of name servers -
    - Recursive Resolver - contacts other DNS servers on behalf of the client.
    - Authoritative Name Server - provides the actual IP address mapping for the domain.

### Resource Records (RRs)

- DNS database stores domain-to-IP mappings in the form of Resource Records (RRs) which is the smallest unit of DNS information.
- Each RR consists of - Type, Name & Value.
- Common Types of Resource Records -

| Type  | Description                                      | Name        | Value          | Example                                                     |
|-------|--------------------------------------------------|-------------|----------------|-------------------------------------------------------------|
| A     | Maps hostname to IP address                      | Hostname    | IP address     | (A, relay1.main.google.com, 104.18.2.119)                 |
| NS    | Defines the authoritative DNS server for a domain| Domain name | Hostname       | (NS, google.com, dns.google.com)                        |
| CNAME | Maps alias to canonical hostname                 | Hostname    | Canonical name | (CNAME, google.com, server1.primary.google.com)         |
| MX    | Defines mail server mapping (alias to canonical) | Hostname    | Canonical name | (MX, mail.google.com, mailserver1.backup.google.com)    |

### Caching

- Temporary storage of frequently requested Resource Records (RRs).
- DNS uses caching at multiple layers -
    - Browser cache
    - Operating System (OS) cache
    - Local resolver cache
    - ISP DNS cache
- Benefits -
    - Reduces latency (faster responses).
    - Reduces query load on DNS infrastructure.
- Even without cache for domain mappings, resolvers may cache TLD or authoritative server addresses to avoid asking the root repeatedly.
- Caching risk - Users may access old IP addresses if cache is stale after updates.
- Mitigation strategies -
    - Use shorter TTL values for critical services.
    - Gradual updates with overlapping infrastructure (old and new servers live simultaneously).
    - Use Content Delivery Networks (CDNs) for smoother propagation.

### Hierarchy

- DNS is not a single server, but a distributed infrastructure of name servers at different levels.
- Ensures scalability for managing billions of queries daily.
- Hierarchical levels -
    - Root servers (.)
    - Top-Level Domain (TLD) servers (e.g., `.com`, `.org`, `.uk`)
    - Authoritative servers (specific to a domain, e.g., `dns.google.com`)
- Four main types of servers -
    - DNS Resolver (Local/Default Server) -
        - Initiates the querying process.
        - Often located within the user’s network (e.g., university, office, ISP).
        - Can cache responses to reduce repeated queries.
    - Root-Level Name Servers -
        - Receive requests from local DNS resolvers.
        - Maintain mappings of Top-Level Domain (TLD) servers (e.g., `.com`, `.edu`, `.us`).
        - Example - If a user requests `google.com`, root servers return the list of TLD servers for `.com`.
    - Top-Level Domain (TLD) Name Servers -
        - Maintain mappings of authoritative servers for domains under the TLD.
        - For `google.com`, the `.com` TLD servers return the authoritative servers for `google.com`.
    - Authoritative Name Servers -
        - The organization’s DNS servers.
        - Provide the actual IP addresses of web/application servers (e.g., `www.google.com` to `142.250.xxx.xxx`).

> [!TIP]
> DNS Resolution Order - DNS processes from right to left (from TLD → subdomain).
> Eg - `www.google.com` - starts from `.com` to `google.com` and then `www.google.com`

### Iterative vs Recursive Query Resolution

- Iterative Query -
    - The local resolver queries step by step: Root → TLD → Authoritative.
    - Each server returns the next server’s address until the final answer is found.
    - Preferred to reduce query load on infrastructure.

- Recursive Query -
    - End user queries the local resolver.
    - The local resolver performs the entire chain of queries (root → TLD → authoritative) and returns the final answer to the user.

### Protocol (UDP over TCP)

- UDP is preferred (faster, less overhead).
- Queries can be retransmitted if needed.
- TCP is avoided due to handshake overhead, except for large DNS responses.

> [!TIP]
> If the network is congested, DNS still prefers UDP for speed, but may retry with TCP if UDP fails.

### Consistency

- DNS compromises strong consistency to achieve performance.
- Provides eventual consistency -
    - Updates propagate slowly (seconds to days).
    - Propagation speed depends on DNS infrastructure, update size, and location in hierarchy.
- Caching can serve outdated (stale) records.
- TTL (Time-To-Live) - Each record has an expiration time to limit staleness.

## Tools

### `nslookup`

- `nslookup www.google.com`
- Non-authoritative answer - provided by cached resolvers (not Google’s authoritative servers).
- Often responses are cached at ISP, university, or office servers.
- Same IP list may appear in different order → DNS enables load balancing.

### `dig`

- `dig www.google.com`
- Shows query time (e.g., 4 ms).
- Shows TTL (e.g., 300 seconds = 5 minutes).
- TTL indicates how long the resolver will keep the mapping cached.