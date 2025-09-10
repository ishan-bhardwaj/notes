# Failure Models in Distributed Systems

- Failures occur in distributed systems in many ways - failure models help reason about their impact and handling.
- Spectrum of different failure models - Fail-stop (easy to deal with) → Crash → Omission → Temporal → Byzantine (difficult to deal with)

## Fail-Stop

- Node halts permanently, but other nodes can detect it by communicating with it.
- Easiest to handle.

## Crash

- Node halts silently - other nodes cannot detect failure immediately.
- Can lead to inconsistencies if undetected.

## Omission Failures

- Node fails to send or receive messages.
- Can cause partial failures and message loss.
- Two types -
    - Send omission - Fails to respond to incoming requests.
    - Receive omission - Fails to receive requests or acknowledge them.

## Temporal Failures

- Node produces correct results but too late to be useful.
- Causes -
    - Poor algorithms
    - Bad design
    - Clock synchronization issues

## Byzantine Failures

- Node behaves arbitrarily (wrong results, random messages, stops midway).
- Hardest to handle.
- Requires Byzantine fault-tolerant algorithms (e.g., PBFT).
- Causes -
    - Malicious attacks
    - Software bugs
