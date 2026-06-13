# Network

## Terminology

| Term | Definition |
|------|------------|
| Interface | Logical instance of a network interface port as seen by the OS — not all are hardware-backed |
| Packet | Message in a packet-switched network — Eg - IP packet |
| Frame | Physical network-level message — Eg - Ethernet frame |
| Socket | BSD API for network endpoints |
| Bandwidth | Maximum rate of data transfer for the network type (bits/s) — Eg - 100 GbE = 100 Gbit/s per direction |
| Throughput | Current data transfer rate between endpoints (bits/s or bytes/s) |
| Latency | Round-trip time between endpoints, or time to establish a connection excluding data transfer |

## Models

### Network Interface

- OS endpoint for network connections — abstraction managed by sysadmins
- Mapped to physical network ports as part of configuration
- Ports have separate transmit and receive channels

### Controller (NIC)

- Provides one or more network ports — houses network controller microprocessor
- Transfers packets between ports and system I/O transport
- Provided as expansion card or built into system board

### Protocol Stack

- Two models — TCP/IP and OSI
- Lower layers encapsulate higher layers — sent messages move down, received messages move up
- Message terminology by OSI layer —
    - Transport layer — segment or datagram
    - Network layer — packet
    - Data link layer — frame
- Additional layers possible — IPsec, WireGuard (above IP), VXLAN tunneling (encapsulates one stack in another)

## Concepts

### Networks and Routing

- Network — group of connected hosts related by protocol addresses
- Smaller subnetworks — isolate broadcasts, make efficient use of network infrastructure
- Routing — manages delivery of packets across networks
- Unicast — transmission between a pair of hosts
- Multicast — sender transmits to multiple destinations simultaneously — must be supported by router config — may be blocked in public cloud
- Shared network components (routers, switches) — contention from other traffic can hurt performance

### Protocols

- Different versions (IPv4, IPv6) may use different kernel code paths — different performance characteristics
- Tunable parameters affect — buffer sizes, algorithms, timers
- Performance-oriented alternatives — SCTP, MPTCP, QUIC

### Encapsulation

- Adds metadata to payload via header, footer, or both
- Increases total message size — small overhead for transmission
- Does not change payload data

### Packet Size

- Larger sizes — higher throughput, fewer packet overheads
- TCP/IP + Ethernet — 54 to 9,054 bytes including headers
- Standard MTU — 1,500 bytes (historical Ethernet balance of NIC buffer cost vs transmission latency)
- Jumbo frames — up to ~9,000 bytes — improves throughput and latency — requires full path support
- Jumbo frame adoption blocked by —
    - Older hardware without jumbo support — fragments or sends ICMP "can't fragment"
    - Misconfigured firewalls blocking ICMP — causes silent packet drops beyond 1,500 bytes
- TSO / GSO — NIC features that narrow the performance gap between 1,500 and 9,000 MTU

### Latency

| Latency Type | Description |
|-------------|-------------|
| Name resolution latency | Time to resolve hostname to IP (DNS) — worst case includes timeouts (tens of seconds) |
| Ping latency | ICMP echo request to response — measures network and kernel stack latency |
| Connection latency | Time from SYN sent to SYN-ACK received — includes kernel TCP session setup and any SYN retransmits |
| First-byte latency (TTFB) | Time from connection established to first data byte received — includes server think time, CPU scheduling, and application processing |
| Round-trip time (RTT) | Time for network request to make full round-trip — dominated by network propagation and hop processing |
| Connection life span | Time from connection established to closed — keep-alive extends this to amortise connection overhead |

### Example Ping Latencies

| From | To | Via | Latency |
|------|-----|-----|---------|
| Localhost | Localhost | Kernel | 0.05 ms |
| Host | Host (same subnet) | 10 GbE | 0.2 ms |
| Host | Host (same subnet) | 1 GbE | 0.6 ms |
| Host | Host (same subnet) | Wi-Fi | 3 ms |
| San Francisco | New York | Internet | 40 ms |
| San Francisco | United Kingdom | Internet | 81 ms |
| San Francisco | Australia | Internet | 183 ms |

### Buffering

- Buffers on sender and receiver sustain high throughput despite network latency
- Larger buffers allow continued sending before blocking on ACKs — important for high-RTT networks
- TCP — sliding send window + buffers
- External buffering (switches, routers) — can cause bufferbloat — long queue intervals — triggers TCP congestion avoidance — throttles performance
- Linux mitigations — byte queue limits, CoDel queueing discipline, TCP small queues
- End-to-end principle — buffering best served by endpoints, not intermediate nodes

### Connection Backlog

- TCP backlog — SYN requests queue in kernel before being accepted by user-land process
- Backlog full — SYN packets dropped — client retransmits — adds latency to connect time
- Backlog drops and SYN retransmits — indicators of host overload
- Tunable — parameter of `listen(2)` syscall and kernel system-wide limits

### Interface Negotiation

- Interfaces autonegotiate bandwidth and duplex with connected transceivers
- May negotiate to lower speeds due to — other endpoint limitations, physical connection problems (bad wiring)
- Full-duplex — bidirectional simultaneous transmission — separate TX/RX paths each at full bandwidth
- Half-duplex — one direction at a time

### Congestion Avoidance

| Protocol | Mechanism |
|----------|-----------|
| Ethernet | Pause frames (IEEE 802.3x) — overwhelmed host requests transmitter to pause — priority pause frames per class |
| IP | Explicit Congestion Notification (ECN) field |
| TCP | Congestion window + pluggable congestion control algorithms |

### Utilization

- Calculated as current throughput / current negotiated bandwidth — per direction
- Full-duplex — utilisation applies to each direction separately
- Servers typically TX-heavy — clients typically RX-heavy
- 100% utilisation in one direction — bottleneck, limits performance
- Tools reporting only packet counts (not bytes) cannot calculate byte-based utilisation — packet size varies greatly

### Local Connections

- Localhost connections use loopback virtual network interface
- IP sockets to localhost — still traverse kernel TCP/IP stack
- Unix Domain Sockets (UDS) — file-based IPC — bypasses TCP/IP stack — lower overhead
- TCP friends (kernel feature) — shortcut TCP/IP stack for localhost data transfers after handshake — not merged
- Cilium BPF — achieves similar shortcut for container networking

## Architecture

### Protocols

#### IP

- IPv4 Type of Service field / IPv6 Traffic Class field — redefined as DSCP + ECN
- DSCP — supports service classes: telephony, broadcast video, low-latency data, high-throughput data, low-priority data
- ECN — allows servers/routers/switches to signal congestion by setting a bit instead of dropping packets — receiver echoes back to sender — sender throttles — avoids packet drop penalty

#### TCP

- Reliable, connection-oriented — RFC 793

##### TCP Performance Features

| Feature | Description |
|---------|-------------|
| Sliding window | Multiple packets up to window size sent before ACK required — window advertised by receiver |
| Congestion avoidance | Prevents sending too much data — avoids saturation and packet drops |
| Slow-start | Begins with small congestion window — increases on ACKs — reduces on timeout |
| SACK | Selective ACK — acknowledges non-contiguous blocks — reduces retransmits |
| Fast retransmit | Retransmits on duplicate ACKs — faster than timer-based |
| Fast recovery | Resets to slow-start after detecting duplicate ACKs |
| TCP Fast Open | Client includes data in SYN — server processes earlier — uses crypto cookie — RFC7413 |
| TCP timestamps | Timestamp in sent packets returned in ACK — enables RTT measurement — RFC 1323 |
| TCP SYN cookies | Cryptographic cookies during SYN floods — legitimate clients continue connecting without server storing state |

##### Three-Way Handshake

- SYN (client) - SYN-ACK (server) - ACK (client)
- Connection latency measured from SYN send to final ACK sent
- Best case — one RTT for handshake
- Packet drop — adds retransmit timeout latency

##### TCP States

- States — `LISTEN`, `SYN-SENT`, `SYN-RECEIVED`, `ESTABLISHED`, `FIN-WAIT-1`, `FIN-WAIT-2`, `CLOSE-WAIT`, `CLOSING`, `LAST-ACK`, `TIME_WAIT`, `CLOSED`
- Performance analysis focuses on `ESTABLISHED` state
- `TIME_WAIT` — fully closed session retained to prevent late packets associating with new connections — typically 2 minutes
- `TIME_WAIT` exhaustion — port exhaustion when connecting frequently to same destination port/IP at high rate (>65,536 connections per 60s)

##### Retransmits

- Timer-based — fires when ACK not received within retransmit timeout (dynamically calculated from RTT)
    - Linux minimum — 200 ms (`TCP_RTO_MIN`) for first retransmit
    - Subsequent — exponential backoff (doubles each time)
- Fast retransmit — triggered by duplicate ACKs — immediate retransmit
- Tail Loss Probe (TLP) — sends probe packet after short timeout on last transmission — detects loss when no subsequent packets trigger duplicate ACK detection

##### Congestion Control Algorithms

| Algorithm | Description |
|-----------|-------------|
| Reno | Triple duplicate ACKs — halve congestion window, halve slow-start threshold, fast retransmit, fast recovery |
| Tahoe | Triple duplicate ACKs — fast retransmit, halve slow-start threshold, set congestion window to 1 MSS, slow-start |
| CUBIC | Cubic function window scaling, hybrid slow-start exit — more aggressive than Reno — Linux default |
| BBR | Explicit network path model using RTT and bandwidth probing — can improve throughput 3x during heavy packet loss (Netflix) — BBRv2 in development |
| DCTCP | Relies on ECN marks from switches at shallow queue occupancy — rapid bandwidth ramp-up — unsuitable for Internet, excellent in controlled DC environments (RFC 8257) |

- Linux 5.6 — supports writing new congestion control algorithms in BPF — load on demand

##### Nagle and Delayed ACKs

- __Nagle (RFC 896)__ — delays small packets to coalesce more data — only delays if data is in pipeline — may conflict with delayed ACKs — disable with `TCP_NODELAY`
- __Delayed ACKs (RFC 1122)__ — delays ACKs up to 500 ms to combine multiple — reduces packets — disable per socket if conflicting with Nagle

##### SACK, FACK, and RACK

- SACK — receiver informs sender of non-contiguous blocks received — avoids retransmitting entire send window on drop
- FACK — extends SACK — tracks additional outstanding data state — better regulates network — Linux default
- RACK-TLP — uses time information from ACKs for loss detection — better than ACK sequences alone — FreeBSD Netflix TCP stack

##### Initial Window (IW)

- Packets sent before waiting for first ACK
- Linux default — 10 packets (IW10) — can be too high on slow links or mass connection startup
- Other OS defaults — IW2 or IW4

#### UDP

- Connectionless, stateless, no retransmits — RFC 768
- Lower overhead than TCP — no congestion avoidance
- Can contribute to network congestion
- Major use — DNS
- Foundation for QUIC

#### QUIC and HTTP/3

- Built on UDP — designed by Google for lower-latency HTTP/TLS
- Features —
    - Multiple application-defined streams multiplexed on same connection
    - Optional reliable in-order stream transport per substream
    - Connection resumption on client network address change (crypto auth)
    - Full payload encryption including QUIC headers
    - 0-RTT connection handshakes for known peers
- IETF standardising QUIC transport and HTTP/3 (HTTP over QUIC)

### Hardware

#### Interfaces

- Physical send/receive of frames on attached network
- Interface types based on Layer 2 standards — each has max bandwidth
- Ethernet speeds — 1 GbE, 10 GbE, 40 GbE, 100 GbE, 200 GbE, 400 GbE (copper or optical)
- Utilisation — current throughput / negotiated bandwidth — per direction (TX, RX separate)
- Wireless — susceptible to poor signal strength and interference

#### Controllers

- Provided via system board or PCIe expander card
- Driven by microprocessors — attached to system via I/O transport (PCIe)
- Either the microprocessor or the PCIe transport can become the throughput bottleneck
- Eg - dual 10 GbE NIC on PCIe Gen2 x4 — NIC max 40 Gbit/s bidirectional, slot max 16 Gbit/s — PCIe is the bottleneck

#### Switches and Routers

- Switches — dedicated communication path between any two connected hosts — no contention (replaced hubs/shared buses)
- Routers — deliver packets between networks using routing tables — path may involve a dozen or more hops
- Paths are dynamic — auto-update on outages and load — actual packet path at any moment is unknown
- Multiple paths — packets may arrive out of order — can cause TCP performance issues
- Switches and routers have their own buffers and CPUs — can bottleneck under load
- Rate transitions (Eg - 10 Gbps to 1 Gbps) at switches — require buffering — over-buffering causes bufferbloat — pacing at source helps

#### Firewalls

- Permit only authorised communications — present as physical devices and kernel software
- Stateful firewalls — store metadata per connection — can hit memory limits under high connection rates (DoS or heavy outbound connections)
- BPF firewalls — growing adoption — Facebook, Cloudflare, Cilium — high performance, programmable, commodity hardware

### Software

#### TCP Connection Queues

- Two backlog queues —
    - SYN backlog — incomplete connections during TCP handshake — first queue — can be long to absorb SYN floods
    - Listen backlog — established sessions waiting for `accept(2)` — second queue
- SYN cookies — bypass first queue — client already authorised
- Both queues tunable independently

#### TCP Buffering

- Send and receive buffers per socket — improve data throughput
- Larger buffers — higher throughput, more memory per connection
- Linux — dynamically increases buffer sizes based on connection activity
- Tunable minimum, default, and maximum sizes

#### Segmentation Offload

- __GSO (Generic Segmentation Offload)__ — kernel sends up to 64 KB "super packets" — split to MSS just before NIC delivery
- __TSO (TCP Segmentation Offload)__ — NIC hardware performs final segmentation — improves kernel stack throughput
- __GRO (Generic Receive Offload)__ — complement to GSO for receive path
- All implemented in kernel software except TSO — NIC hardware

#### Queueing Disciplines (qdiscs)

- Optional layer for traffic classification, scheduling, manipulation, filtering, shaping
- Configured via `tc(8)` — each qdisc has its own man page
- Linux default — `pfifo_fast` (kernel) or `fq_codel` (systemd)
- BPF programs — `BPF_PROG_TYPE_SCHED_CLS` and `BPF_PROG_TYPE_SCHED_ACT` — attach to kernel ingress/egress for filtering, mangling, forwarding

#### Network Device Drivers

- Ring buffer — additional buffer between kernel memory and NIC for TX/RX
- Interrupt coalescing — interrupt sent only after timer or packet count threshold — higher throughput, slightly higher latency
- NAPI framework —
    - Low packet rates — interrupts used (softirq scheduling)
    - High packet rates — interrupts disabled, polling used — allows coalescing
    - Packet throttling — early drop at NIC to prevent system overwhelm
    - Interface scheduling — quota per polling cycle — fairness between busy interfaces
    - `SO_BUSY_POLL` — user-space apps can busy-wait on socket to reduce receive latency

#### CPU Scaling

| Mechanism | Description |
|-----------|-------------|
| RSS (Receive Side Scaling) | NIC hardware hashes packets to multiple queues — different CPUs process different queues — interrupts CPU directly — hash on IP + TCP port |
| RPS (Receive Packet Steering) | Software RSS for NICs without multiple queues — short ISR maps packet to CPU for processing |
| RFS (Receive Flow Steering) | RPS with affinity for CPU where socket was last processed — improves CPU cache hit rates |
| Accelerated RFS | RFS in NIC hardware — NIC updated with flow info to determine which CPU to interrupt |
| XPS (Transmit Packet Steering) | NICs with multiple TX queues — multi-CPU transmission |

- Without CPU load balancing — single CPU handles all NIC interrupts — 100% softirq on one CPU — visible via `mpstat`
- `irqbalance` process — assigns IRQ lines to CPUs based on cache coherency

#### Kernel Bypass

- DPDK — application implements network protocols in user-space — direct NIC memory access — bypasses kernel stack entirely
- XDP (eXpress Data Path) — BPF programmable fast path — integrates into existing kernel stack — does not bypass it
- Bypass consequence — traditional tools and metrics unavailable — counters and tracing events also bypassed
- Zero-copy alternatives — `MSG_ZEROCOPY` send flag, `mmap(2)` receive

#### TCP Send Path Optimizations

| Component | Description |
|-----------|-------------|
| Pacing | Controls when to send packets — spreads transmissions — avoids bursts that cause queueing delay or switch drops |
| TSQ (TCP Small Queues) | Controls and reduces amount queued by network stack — avoids bufferbloat |
| BQL (Byte Queue Limits) | Auto-sizes driver queues — large enough to avoid starvation, small enough to reduce max latency and avoid NIC TX descriptor exhaustion — added Linux 3.3 |
| EDT (Earliest Departure Time) | Timing wheel instead of queue for NIC packet ordering — timestamps set on every packet per policy — added Linux 4.20 |

## Methodology

### Tools Method

```bash
nstat                    # check retransmit rate and out-of-order packets
netstat -s               # network stack statistics
ip -s link               # interface error counters — errors, dropped, overruns
ss -tiepm                # per-socket limiter flag, RTT, buffer usage, congestion details
nicstat                  # bytes TX/RX per interface with utilisation
tcplife                  # TCP session log with process, duration, throughput
tcptop                   # top TCP sessions live
tcpdump                  # short captures for unusual traffic or protocol inspection
perf / BCC / bpftrace    # inspect packets and kernel state between application and wire
```

### USE Method

- Per network interface, per direction (TX and RX) —
    - __Utilisation__ — current throughput / current negotiated bandwidth — check both directions
    - __Saturation__ — extra queueing, buffering, or blocking due to fully utilised interface — difficult to measure directly — check time application threads block on network sends, Linux "overruns" counter
    - __Errors__ — RX: bad checksum, short/long frames, collisions — TX: late collisions (bad wiring)
- Check errors first — quick and easiest to interpret
- TCP retransmits — indicator of network saturation — but measured across full network path, not just one interface
- Controller utilisation — sum interface throughputs for all ports on same controller — compare to known max

### Workload Characterization

- Basic metrics —
    - Network interface throughput — RX and TX (bytes/s) per interface
    - Network interface IOPS — RX and TX (frames/s)
    - TCP connection rate — active (connect) and passive (accept) connections/s

#### Advanced Checklist

- Average packet size — RX, TX
- Protocol breakdown per layer — TCP vs UDP (including QUIC)
- Active TCP/UDP ports — bytes/s and connections/s per port
- Broadcast and multicast packet rates
- Processes actively using network

### Latency Analysis

| Latency | Measurement Approach |
|---------|---------------------|
| Name resolution | DNS lookup timing — check caching, TTL, server proximity |
| Ping | ICMP RTT — `ping(8)` — may not match application RTT (ICMP priority varies) |
| TCP connection init | SYN to SYN-ACK time — exercises kernel TCP stack |
| TCP first-byte (TTFB) | Connection to first data byte — includes server think time and scheduling |
| TCP retransmits | Present at all — add hundreds to thousands of ms — check `netstat -s` / `tcpretrans` |
| TIME_WAIT | Duration locally closed sessions wait — port exhaustion risk at high connection rates |
| Connection lifespan | Total duration — keep-alive amortises connection overhead |
| System call latency | Socket read/write syscall time |
| Inter-stack latency | Time for packet to traverse kernel TCP/IP stack |

- Present as — per-interval averages, full distributions (histograms/heat maps), per-operation with source/destination details
- Transient outliers — only appear under load — measure both idle and loaded system
- Socket options for timestamps — `SO_TIMESTAMP`, `SO_TIMESTAMPNS`, `SO_TIMESTAMPING`

### Performance Monitoring

- Key metrics to monitor over time —
    - Throughput — RX and TX bytes/s per interface
    - TCP connections/s
    - Error counters including dropped packets
    - TCP retransmits — correlate with network issues
    - TCP out-of-order packets

### Packet Sniffing

- Last resort — expensive in CPU and storage
- Use ring buffers to reduce overhead — shared memory between kernel and user-level tool
- Out-of-band sniffer — separate server on switch tap/mirror port — no overhead on production host
- Kernel BPF filtering — expression compiled to BPF bytecode, JIT-compiled — filters in kernel before transferring to user space
- Capture to file — then analyse offline — practical at high packet rates

### TCP Analysis

- TCP send/receive buffer usage
- TCP backlog queue usage
- Kernel drops from full backlog queue
- Congestion window size including zero-size advertisements
- SYNs during TIME_WAIT — port exhaustion risk
- TIME_WAIT exhaustion — same src/dst IP + dst port, different ephemeral src port — 16-bit port space + TIME_WAIT duration = limit of ~65,536 connections in 60s
- Mitigations — `tcp_tw_reuse`, multiple server IPs, `SO_LINGER` with low linger time

### Static Performance Tuning

- Configuration checklist —
    - Number of interfaces available and in use
    - Maximum and currently negotiated speed per interface
    - Half vs full duplex negotiation
    - MTU configured per interface
    - Interface trunking configured
    - Tunable parameters changed from defaults — IP, TCP, device driver
    - Routing configuration and default gateway
    - Maximum throughput of all network components in data path (switch/router backplanes)
    - Maximum MTU for full path — fragmentation occurring
    - Wireless links in data path — interference
    - Forwarding enabled (acting as router)
    - DNS server location and configuration
    - Known firmware/driver performance bugs
    - Firewalls present and their rule complexity
    - Software-imposed throughput limits (resource controls)

### Resource Controls

| Control Type | Description |
|-------------|-------------|
| Network bandwidth limits | Maximum throughput per protocol or application — enforced by kernel |
| IP QoS (ToS/DSCP) | Traffic prioritisation by network components — IP header ToS bits — Differentiated Services |
| Packet latency (`tc-netem`) | Adds artificial latency — useful for testing and simulation |

## Observability Tools

## ss

- Socket statistics tool — shows open sockets with detailed per-socket metrics
- Source — netlink(7) interface (binary, efficient) vs `netstat` which uses `/proc/net` text files

```bash
ss                      # all open sockets
ss -t                   # TCP sockets only
ss -tiepm               # TCP + internal info + extended + process + memory
```

### Key ss Output Fields

| Field | Description |
|-------|-------------|
| `rto` | TCP retransmission timeout (ms) |
| `rtt` | Average RTT / mean deviation (ms) |
| `mss` | Maximum segment size (bytes) |
| `cwnd` | Congestion window size (× MSS) |
| `bytes_acked` | Total bytes successfully transmitted |
| `bytes_received` | Total bytes received |
| `pacing_rate` | Current pacing rate (Mbps) |
| `minrtt` | Minimum RTT observed — compare to average for congestion indication |
| `bbr:(bw, mrtt, ...)` | BBR congestion control stats |

- Limiter flags —
    - `app_limited` — congestion window not fully utilised — application is the bottleneck
    - `rwnd_limited:Xms` — limited by receiver window — includes time limited
    - `sndbuf_limited:Xms` — limited by send buffer — includes time limited

> [!TIP]
> To find connection age (not shown by `ss`), use `stat /proc/<PID>/fd/<FD>` and check the change timestamp on the file descriptor.

## ip

- Manages routing, network devices, interfaces, tunnels — primary iproute2 tool

```bash
ip -s link              # interface stats — RX/TX bytes, packets, errors, drops, overruns, collisions
ip -s -s link           # even more error type details
ip route                # routing table
ip monitor              # watch netlink messages in real time
```

### Interface Error Counters

| Counter | Description |
|---------|-------------|
| `errors` (RX) | Receive errors |
| `dropped` (RX/TX) | Dropped packets |
| `overrun` (RX) | Receive overruns |
| `errors` (TX) | Transmit errors |
| `carrier` (TX) | Carrier errors |
| `collsns` (TX) | Collisions — rare on switched networks, indicates errors if non-zero |

> [!NOTE]
> `ip` shows cumulative counters since interface was brought UP — not per-interval rates. Use `sar -n DEV` for rates.

## ifconfig

- Traditional interface administration tool — considered obsolete on Linux, replaced by `ip(8)`
- Same error counters as `ip` — RX/TX errors, drops, overruns, frame errors, carrier, collisions

## nstat

- Prints network metrics with SNMP names — from `/proc/net/snmp` and `/proc/net/netstat`

```bash
nstat -s          # print without resetting counters
nstat             # print and reset counters (useful for before/after command comparison)
nstat -rs         # restore counters to since-boot values after accidental reset
nstat -d          # daemon mode — collect interval statistics
```

### Key nstat Metrics

| Metric | Description |
|--------|-------------|
| `IpInReceives` | Inbound IP packets |
| `IpOutRequests` | Outbound IP packets |
| `TcpActiveOpens` | TCP active connections (`connect(2)`) |
| `TcpPassiveOpens` | TCP passive connections (`accept(2)`) |
| `TcpInSegs` | TCP inbound segments |
| `TcpOutSegs` | TCP outbound segments |
| `TcpRetransSegs` | TCP retransmitted segments — compare ratio vs `TcpOutSegs` |

## netstat

- Multi-tool for network statistics — considered deprecated on Linux

```bash
netstat -s          # network stack statistics
netstat -i          # interface statistics
netstat -r          # route table
netstat -an         # all sockets, no hostname resolution
netstat -i -c 1     # continuous interface counter output every second
```

### Interface Columns (`-i`)

| Column | Description |
|--------|-------------|
| `RX-OK` | Packets received successfully |
| `RX-ERR` | Receive errors |
| `RX-DRP` | Receive drops — indicates interface saturation |
| `RX-OVR` | Receive overruns — indicates interface saturation |
| `TX-OK` | Packets transmitted successfully |
| `TX-ERR` | Transmit errors |
| `TX-DRP` | Transmit drops |
| `TX-OVR` | Transmit overruns |

### Key TCP Stack Statistics (`-s`) to Watch

| Statistic | Significance |
|-----------|-------------|
| High forwarded vs total packets received | Check whether server is supposed to be routing |
| High retransmits vs segments sent | Unreliable network |
| `TCPSynRetrans` | Server dropping SYNs from full listen backlog |
| `Packets pruned from receive queue because of socket buffer overrun` | Network saturation — increase socket buffers |

> [!TIP]
> For reliable SNMP-named stats, prefer `nstat` over `netstat -s`. For raw data, read `/proc/net/snmp` and `/proc/net/netstat` directly.

## sar

- Historical network statistics — `-n` option with various sub-options

```bash
sar -n DEV 1        # network interface stats per second
sar -n EDEV 1       # interface error stats
sar -n TCP 1        # TCP stats
sar -n ETCP 1       # TCP error stats
sar -n IP 1         # IP datagram stats
sar -n SOCK 1       # socket usage
sar -n DEV 1 | awk 'NR == 3 || $2 == "ens5"'   # filter to specific interface
```

### sar Network Statistics Reference

| Option | Statistic | Description | Units |
|--------|-----------|-------------|-------|
| `-n DEV` | `rxpkt/s`, `txpkt/s` | Received/transmitted packets | Packets/s |
| `-n DEV` | `rxkB/s`, `txkB/s` | Received/transmitted kilobytes | KB/s |
| `-n DEV` | `%ifutil` | Interface utilisation — greater of RX or TX for full duplex | % |
| `-n EDEV` | `rxerr/s`, `txerr/s` | Receive/transmit packet errors | Packets/s |
| `-n EDEV` | `rxdrop/s`, `txdrop/s` | Received/transmitted dropped packets | Packets/s |
| `-n EDEV` | `rxfifo/s`, `txfifo/s` | FIFO overrun errors | Packets/s |
| `-n TCP` | `active/s` | New active TCP connections (`connect(2)`) | Connections/s |
| `-n TCP` | `passive/s` | New passive TCP connections (`accept(2)`) | Connections/s |
| `-n TCP` | `iseg/s`, `oseg/s` | Input/output segments | Segments/s |
| `-n ETCP` | `retrans/s` | TCP segments retransmitted | Segments/s |
| `-n ETCP` | `estres/s` | Established connection resets | Resets/s |
| `-n SOCK` | `tcpsck` | Total TCP sockets in use | Sockets |
| `-n SOCK` | `tcp-tw` | TCP sockets in TIME_WAIT | Sockets |

## nicstat

- Network interface throughput and utilisation — follows `iostat`/`mpstat` style
- Provides utilisation and saturation values — particularly useful for USE method

```bash
nicstat -z 1        # 1-second interval, skip zero-activity interfaces
nicstat -t 1        # include TCP statistics
```

| Column | Description |
|--------|-------------|
| `rKB/s`, `wKB/s` | Receive/transmit KB/s |
| `rPk/s`, `wPk/s` | Receive/transmit packets/s |
| `rAvs`, `wAvs` | Average receive/transmit packet size (bytes) |
| `%Util` | Maximum utilisation — greater of RX or TX |
| `Sat` | Saturation value |

## ethtool

- Network interface driver statistics, configuration, and tunables

```bash
ethtool -S eth0     # driver statistics (device-specific — varies by driver)
ethtool -i eth0     # driver name and version
ethtool -k eth0     # interface feature tunables (TSO, GSO, GRO, checksumming, etc.)
ethtool -K eth0 tso on   # enable TCP segmentation offload
```

## tcplife

- BCC and bpftrace tool — traces TCP session lifespan — prints on session close

```bash
tcplife             # all TCP sessions
tcplife -t          # include HH:MM:SS timestamp
tcplife -w          # wider columns for IPv6
tcplife -p PID      # trace single process
tcplife -L PORT     # filter by local port
tcplife -D PORT     # filter by remote port
```

| Column | Description |
|--------|-------------|
| `PID` | Process ID |
| `COMM` | Process name |
| `LADDR`, `LPORT` | Local address and port |
| `RADDR`, `RPORT` | Remote address and port |
| `TX_KB`, `RX_KB` | Bytes transmitted/received (KB) |
| `MS` | Session duration (milliseconds) |

- Traces TCP socket state-change events — prints on `TCP_CLOSE`
- Much lower overhead than per-packet sniffers — acceptable for continuous production logging

## tcptop

- BCC tool — top TCP sessions by throughput — updates every second

```bash
tcptop              # all TCP, 1-second refresh
tcptop -C           # don't clear screen
tcptop -p PID       # single process
```

| Column | Description |
|--------|-------------|
| `PID`, `COMM` | Process ID and name |
| `LADDR`, `RADDR` | Local and remote address with port |
| `RX_KB`, `TX_KB` | Bytes received/transmitted during interval (KB) |

> [!NOTE]
> `tcptop` traces TCP send/receive code path — high network throughput systems may have measurable overhead. Use for investigation, not continuous monitoring.

## tcpretrans

- BCC and bpftrace tool — traces TCP retransmit events with TCP state

```bash
tcpretrans          # trace all retransmits
tcpretrans -l       # include tail loss probe attempts
tcpretrans -c       # count retransmits per flow (summary instead of per-event)
```

| Column | Description |
|--------|-------------|
| `TIME` | Timestamp |
| `PID` | Process ID |
| `IP` | IP version (4 or 6) |
| `LADDR:LPORT` | Local address and port |
| `T>` | Type — R = retransmit |
| `RADDR:RPORT` | Remote address and port |
| `STATE` | TCP state at time of retransmit |

- High rate in `ESTABLISHED` — external network problem
- High rate in `SYN_SENT` — overloaded server not consuming SYN backlog fast enough
- Negligible overhead — retransmits are infrequent events
- Advantage over packet capture — prints TCP state directly from kernel without capturing all packets

## bpftrace

- BPF-based custom network tracing

### One-Liners

```bash
# Count socket accepts by PID and process
bpftrace -e 't:syscalls:sys_enter_accept* { @[pid, comm] = count(); }'

# Count socket connects by PID and process
bpftrace -e 't:syscalls:sys_enter_connect { @[pid, comm] = count(); }'

# Count socket connects by user stack trace
bpftrace -e 't:syscalls:sys_enter_connect { @[ustack, comm] = count(); }'

# Count socket send/receives by direction, PID, process
bpftrace -e 'k:sock_sendmsg,k:sock_recvmsg { @[func, pid, comm] = count(); }'

# Count socket send/receive bytes by PID and process
bpftrace -e 'kr:sock_sendmsg,kr:sock_recvmsg /(int32)retval > 0/ { @[pid, comm] = sum((int32)retval); }'

# Count TCP connects by PID and process
bpftrace -e 'k:tcp_v*_connect { @[pid, comm] = count(); }'

# Count TCP accepts by PID and process
bpftrace -e 'k:inet_csk_accept { @[pid, comm] = count(); }'

# TCP send bytes histogram
bpftrace -e 'k:tcp_sendmsg { @send_bytes = hist(arg2); }'

# TCP receive bytes histogram
bpftrace -e 'kr:tcp_recvmsg /retval >= 0/ { @recv_bytes = hist(retval); }'

# Count TCP retransmits by type and remote host (IPv4)
bpftrace -e 't:tcp:tcp_retransmit_* { @[probe, ntop(2, args->saddr)] = count(); }'

# Count all TCP functions (high overhead)
bpftrace -e 'k:tcp_* { @[func] = count(); }'

# UDP send bytes histogram
bpftrace -e 'k:udp_sendmsg { @send_bytes = hist(arg2); }'

# Count transmit kernel stack traces
bpftrace -e 't:net:net_dev_xmit { @[kstack] = count(); }'

# Show receive CPU histogram per device
bpftrace -e 't:net:netif_receive_skb { @[str(args->name)] = lhist(cpu, 0, 128, 1); }'

# Count all ixgbevf device driver functions
bpftrace -e 'k:ixgbevf_* { @[func] = count(); }'
```

### Socket Tracing

```bash
# Count accepts by process
bpftrace -e 't:syscalls:sys_enter_accept { @[pid, comm] = count(); }'

# Count accepts by user stack trace — identify code path
bpftrace -e 't:syscalls:sys_enter_accept { @[ustack, comm] = count(); }'
```

### sock Tracepoints

```bash
bpftrace -l 't:sock:*'
# tracepoint:sock:sock_rcvqueue_full
# tracepoint:sock:sock_exceed_buf_limit
# tracepoint:sock:inet_sock_set_state
```

```bash
# Count new IPv4 connections by source and destination address
bpftrace -e 't:sock:inet_sock_set_state
    /args->newstate == TCP_ESTABLISHED && args->family == AF_INET/ {
    @[ntop(args->saddr), ntop(args->daddr)] = count() }'
```

### TCP Tracepoints

```bash
bpftrace -l 't:tcp:*'
# tcp_retransmit_skb, tcp_send_reset, tcp_receive_reset
# tcp_destroy_sock, tcp_rcv_space_adjust, tcp_retransmit_synack, tcp_probe
```

### TCP SYN Backlog (tcpsynbl.bt)

```bpftrace
kprobe:tcp_v4_syn_recv_sock,
kprobe:tcp_v6_syn_recv_sock
{
    $sock = (struct sock *)arg0;
    @backlog[$sock->sk_max_ack_backlog & 0xffffffff] =
        hist($sock->sk_ack_backlog);
    if ($sock->sk_ack_backlog > $sock->sk_max_ack_backlog) {
        time("%H:%M:%S dropping a SYN.\n");
    }
}
```

- Prints SYN drops as they occur — histogram of backlog length per limit on exit
- Shows how close to overflow — useful for capacity planning

### Network Event Sources

| Network Event | Event Source |
|---------------|-------------|
| Application protocols | uprobes |
| Sockets | syscalls tracepoints |
| TCP | tcp tracepoints, kprobes |
| UDP | kprobes |
| IP and ICMP | kprobes |
| Packets | skb tracepoints, kprobes |
| QDiscs and driver queues | qdisc and net tracepoints, kprobes |
| XDP | xdp tracepoints |
| Network device drivers | kprobes, some have tracepoints |

## tcpdump

- Network packet capture — print to stdout or write to file

```bash
tcpdump -ni eth4                    # capture from interface, no hostname resolution
tcpdump -i eth4 -w /tmp/out.tcpdump  # capture to file
tcpdump -nr /tmp/out.tcpdump         # read and print from file
tcpdump -i any                       # capture from all interfaces
tcpdump -enr file -vvv -X            # ethernet headers, verbose, hex dump
tcpdump -ttt -r file                 # delta times between packets
tcpdump -ttttt -r file               # elapsed time since first packet
tcpdump -i eth0 'port 80'            # filter expression (compiled to BPF in kernel)
```

- Drop count reported at end — packets dropped by kernel when rate too high
- Filter expression — compiled to BPF bytecode, JIT-compiled — filtering in kernel reduces overhead
- Use for short periods only — expensive in CPU and storage

> [!NOTE]
> Prefer BPF-based tools (`bpftrace`, `tcpretrans`, `tcplife`) over `tcpdump` for performance investigation — much lower overhead and richer kernel context.

## Wireshark

- Graphical interface for packet capture and inspection — imports `tcpdump` dump files
- Three-pane view — packet table / protocol detail tree / hex dump
- Identifies network connections and their related packets for separate study
- Translates hundreds of protocol headers

## Other Tools

| Tool | Description |
|------|-------------|
| `sockstat` | High-level socket statistics |
| `sofamily` | Count address families for new sockets by process |
| `soprotocol` | Count transport protocols for new sockets by process |
| `soconnect` | Trace socket IP-protocol connections with details |
| `soaccept` | Trace socket IP-protocol accepts with details |
| `socketio` | Summarise socket details with I/O counts |
| `socksize` | Socket I/O sizes as per-process histograms |
| `sormem` | Socket receive buffer usage and overflows |
| `soconnlat` | IP socket connection latency with stacks |
| `so1stbyte` | IP socket first-byte latency |
| `tcpconnect` | Trace TCP active connections (connect) |
| `tcpaccept` | Trace TCP passive connections (accept) |
| `tcpwin` | Trace TCP send congestion window parameters |
| `tcpnagle` | Trace TCP Nagle usage and transmit delays |
| `gethostlatency` | DNS lookup latency via library calls |
| `ipecn` | Trace IP inbound ECN |
| `skbdrop` | Trace sk_buff drops with kernel stack traces |
| `skblife` | sk_buff lifespan as inter-stack latency |
| `strace` | Trace socket-related syscalls (high overhead) |
| `lsof` | List open files by PID including socket details |
| `/proc/net` | Contains many network statistics files |

## Experimentation

### ping

```bash
ping www.netflix.com    # ICMP echo request/response — RTT per packet + statistics
```

- Newer kernels and `ping` use `SO_TIMESTAMP` for kernel-level timestamps — more accurate
- ICMP may be lower priority on routers — higher latency variance than application traffic

### traceroute

```bash
traceroute www.example.com    # ICMP TTL-based route discovery
traceroute -T www.example.com  # use TCP instead of ICMP (workaround for ICMP-blocking firewalls)
```

- Sends packets with increasing TTL — each hop responds with ICMP time exceeded
- Three RTT measurements per hop — `*` = no ICMP returned or ICMP blocked
- Path can change during run — different addresses on same hop line indicates multi-path
- Use for static performance tuning — verify path and detect degraded routes

### iperf

```bash
# Server
iperf -s -l 128k                          # listen with 128 KB socket buffer

# Client
iperf -c <host> -l 128k -P 2 -i 1 -t 60  # connect, 128 KB buffer, 2 parallel threads, 1s interval, 60s
```

- High-bandwidth interfaces (100 Gbps) may require multiple parallel threads (`-P`) to reach line rate
- `--reportstyle C` — CSV output for graphing

### netperf

```bash
netserver -D -p 7001                                      # start server
netperf -v 100 -H <host> -t TCP_RR -p 7001               # measure TCP round-trip latency
```

- `TCP_RR` mode — request/response — measures round-trip latency
- More accurate for RTT measurement than `ping`

### tc (Traffic Control)

```bash
tc qdisc show dev eth0                    # show current qdisc
tc qdisc add dev eth0 root netem loss 1%  # add 1% packet loss
tc -s qdisc show dev eth0                 # show qdisc with drop statistics
tc qdisc del dev eth0 root                # remove qdisc
```

- `netem` — network emulator qdisc — simulate packet loss, delay, corruption, duplication
- Useful for testing application behaviour under degraded network conditions

## Tuning

### System-Wide (`sysctl`)

```bash
sysctl -a | grep tcp    # list all TCP tunables (70+ on kernel 5.3)
```

### Netflix Production Example

```bash
net.core.default_qdisc = fq
net.core.netdev_max_backlog = 5000
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.somaxconn = 1024
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_abort_on_overflow = 1
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_rmem = 4096 12582912 16777216
net.ipv4.tcp_wmem = 4096 12582912 16777216
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_tw_reuse = 1
```

### Key Tunables Reference

| Tunable | Description |
|---------|-------------|
| `net.core.rmem_max`, `net.core.wmem_max` | Maximum socket buffer for all protocols — set to 16 MB+ for 10 GbE |
| `net.ipv4.tcp_rmem`, `net.ipv4.tcp_wmem` | TCP buffer auto-tuning — min/default/max bytes |
| `net.ipv4.tcp_moderate_rcvbuf = 1` | Enable TCP receive buffer auto-tuning |
| `net.ipv4.tcp_max_syn_backlog` | First backlog queue — half-open connections — increase for burst load |
| `net.core.somaxconn` | Second backlog queue — established connections waiting for `accept(2)` |
| `net.core.netdev_max_backlog` | Network device backlog queue per CPU — increase to 10000 for 10 GbE |
| `net.ipv4.tcp_congestion_control` | System-wide default congestion control algorithm |
| `net.ipv4.tcp_sack = 1` | Enable selective ACKs — improves throughput on high-latency networks |
| `net.ipv4.tcp_fack = 1` | Enable FACK extensions — better outstanding data regulation |
| `net.ipv4.tcp_tw_reuse = 1` | Allow TIME_WAIT session reuse when safe — higher connection rates between same pair |
| `net.ipv4.tcp_tw_recycle = 0` | Disable — less safe than `tcp_tw_reuse` |
| `net.ipv4.tcp_ecn` | 0 = disable ECN, 1 = request on outgoing, 2 = allow on incoming only |
| `net.ipv4.ip_local_port_range` | Ephemeral port range — expand to increase max outbound connections |

```bash
# Add and load a new congestion control module
modprobe tcp_htcp
sysctl net.ipv4.tcp_available_congestion_control   # verify available algorithms
```

### Socket Options (setsockopt)

| Option | Description |
|--------|-------------|
| `SO_SNDBUF`, `SO_RCVBUF` | Send/receive buffer sizes (up to system limits) |
| `SO_REUSEPORT` | Multiple processes/threads bind same port — kernel distributes load — Linux 3.9+ |
| `SO_MAX_PACING_RATE` | Maximum pacing rate (bytes/s) |
| `SO_LINGER` | Reduce TIME_WAIT latency |
| `SO_TXTIME` | Time-based packet transmission with deadlines — Linux 4.19+ |
| `TCP_NODELAY` | Disable Nagle — send segments immediately — lower latency, more packets |
| `TCP_CORK` | Pause until full packets — higher throughput |
| `TCP_QUICKACK` | Send ACKs immediately — increases send bandwidth |
| `TCP_CONGESTION` | Per-socket congestion control algorithm override |
| `MSG_ZEROCOPY` | Send flag — use user-space buffer directly — avoids kernel copy — Linux 4.14+ |

### Configuration Options

- Jumbo frames — increase MTU from 1,500 to ~9,000 — requires full path support including switch config
- Link aggregation — group interfaces for combined bandwidth — requires switch support
- Firewall `iptables` / BPF — set IP ToS (DSCP) based on port rules — traffic prioritisation

### Byte Queue Limits

```bash
# View per-queue BQL limits
grep . /sys/devices/pci.../net/ens5/queues/tx-0/byte_queue_limits/limit*
# limit (auto-tuned), limit_max, limit_min

# Clamp auto-tuning range
echo 0 > /sys/.../byte_queue_limits/limit_min
echo 32768 > /sys/.../byte_queue_limits/limit_max
```

### cgroups Network Controls

| cgroup Subsystem | Description |
|-----------------|-------------|
| `net_prio` | Priority for outgoing traffic per process group |
| `net_cls` | Tag packets with class ID — used by qdiscs and BPF programs for limits |

### Queueing Disciplines

```bash
sysctl net.core.default_qdisc             # view current default
sysctl -w net.core.default_qdisc=fq_codel  # set default
man -k tc-                                 # list all available qdiscs
```

- `fq_codel` — default on most distributions — good general performance, avoids bufferbloat
- `fq` — fair queueing — used by Netflix

### Tuned Project

```bash
tuned-adm list                            # list available profiles
tuned-adm profile network-latency         # activate low-latency profile
tuned-adm profile network-throughput      # activate throughput profile
```

- `network-latency` profile sets — `net.core.busy_read=50`, `net.core.busy_poll=50`, `net.ipv4.tcp_fastopen=3`, `kernel.numa_balancing=0`