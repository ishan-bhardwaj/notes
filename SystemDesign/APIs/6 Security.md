# API Security

- **Major Attack Surfaces** -
  - DDoS - Botnets overwhelm services with traffic.
  - Insufficient Authentication - Malicious calls bypass identity verification.
  - Injections - XSS/SQLi execute unauthorized code.
  - Data Exposure - Man-in-the-middle intercepts unencrypted transit data.

- **Common API Attacks** -
| Attack                     | Description                                       | Impact                    | Attacker Motivation                                      |
| -------------------------- | ------------------------------------------------- | ------------------------- | -------------------------------------------------------- |
| Code injection             | Executes injected code to alter API functionality | System-wide compromise    | Denial of service, data theft, financial loss educative​ |
| Cross-site scripting (XSS) | Runs malicious scripts in client browsers         | Browser session hijacking | Steal user data via scripts educative​                   |
| SQL injection              | Modifies database queries through inputs          | Database manipulation     | Data theft, deletion, or alteration educative​           |
| CRLF injection             | Splits HTTP headers to alter responses            | Response tampering        | Data alteration, similar to XSS educative​               |
| Buffer overflow            | Overloads memory with excessive input             | System corruption         | Execution path changes, info exposure educative​         |

- **CIA Triad Fundamentals** -
| Principle       | Goal                   | Protection                            |
| --------------- | ---------------------- | ------------------------------------- |
| Confidentiality | Authorized access only | Encryption (TLS), Authenticationkmcd​ |
| Integrity       | Prevent tampering      | Input validation, checksums           |
| Availability    | Reliable uptime        | Rate limiting, DDoS mitigationkmcd​   |

- **STRIDE Threat Model** -
| Threat                 | Description              | Countermeasure                                  |
| ---------------------- | ------------------------ | ----------------------------------------------- |
| Spoofing               | Identity impersonation   | Authentication tokens                           |
| Tampering              | Data alteration          | Validation + signatures                         |
| Repudiation            | Deny actions             | Audit logging                                   |
| Information Disclosure | Data leaks               | Encryption + access control                     |
| Denial of Service      | Resource exhaustion      | Rate limiting                                   |
| Elevation of Privilege | Unauthorized access gain | Least privilege principlepractical-devsecops+1​ |

- **Security Process Flow** -
  - Identify Assets - Databases, servers, user data
  - Define CIA Goals - Map protection requirements
  - Threat Modeling - STRIDE analysis on data flow diagrams with trust boundaries
  - Implement Mechanism - Encryption, validation, auth/authz

## Transport Layer Security (TLS)

- Transport Layer Security (TLS) provides cryptographic protection for API client-server communications, ensuring confidentiality, integrity, authenticity, and non-repudiation via encryption and certificates.
- **Goal** -
  - Unsecured transmissions (e.g., public Wi-Fi) expose requests/responses to interception, disclosure, and tampering. 
  - TLS addresses - server identity verification, message secrecy, integrity checks, and authenticity proofs.
​
- **TLS Handshake Flow** -
  - Client Hello - TLS version, client random, supported ciphers
  - Server Hello - Chosen cipher, server random, certificate (public key), signed hash (digital signature)
  - Certificate Verification - Client validates via CA
  - Premaster Secret - Client generates, encrypts with server public key; both derive symmetric session key (client random + server random + premaster)
  - Finished Messages - Encrypted with shared key; confirm handshake

> [!TIP]
> Client Random + Server Random + Premaster Secret → Symmetric Key (session-specific)

- **Data Integrity (HMAC)** -
  - Append HMAC (hash + secret key) to message
  - Encrypt full payload (message + HMAC) with symmetric key
  - Receiver decrypts, recomputes HMAC, compares → detects tampering

### HTTP vs HTTPS

| Protocol | Security         | Performance      | Data Form            |
| -------- | ---------------- | ---------------- | -------------------- |
| HTTP     | None (plaintext) | Faster           | Readable             |
| HTTPS    | TLS encrypted    | Slower handshake | Ciphertexteducative​ |

- **Key Properties Achieved** -
  - Server Auth - Digital certificates + signatures
  - Confidentiality - Symmetric encryption
  - Integrity - HMAC verification
  - Authenticity - Signatures prevent replay attacks
​
- **Best Practices** -
  - Mandate TLS 1.3 (fewer round trips, secure ciphers)
  - Use TLS False Start (if compatible)
  - HSTS headers (force HTTPS)
  - Perfect Forward Secrecy (ephemeral keys)
  - Regular cipher suite updates[web:96]

- **TLS vs SSL** -
  - SSL deprecated; TLS is successor (v1.0+). TLS 1.3 drops MD5/SHA1, supports 0-RTT resumption.
​
## Input Validation

- **Client Side Validation** -
  - Client-side checks occur in the browser using JavaScript or HTML5 form validation, providing instant user feedback. 
  - They highlight errors (e.g., invalid email format) and reduce server load by rejecting bad data early. 
  - However, attackers can bypass them via tools like browser dev consoles or direct HTTP requests.

- **Server Side Validation** -
  - Server validation acts as the definitive check after client-side filtering, using multiple techniques -
    - Format check - Ensures correct patterns (e.g., no integers in names).
    - Length check - Limits sizes (e.g., 3-letter country codes).
    - Presence check: Requires mandatory fields (e.g., non-empty username).
    - Range check - Bounds values (e.g., `age ≥ 0`).
    - Lookup table - Matches predefined lists (e.g., valid days: Mon-Sun).
    - Harmful character removal - Strips injection risks like `%` or `<script>`.

> [!TIP]
> Implement both layers for defense-in-depth, rejecting invalid requests with HTTP 400 and generic error messages to avoid leaking details.

## Cross-Origin Resource Sharing (CORS)

- CORS enables secure cross-origin HTTP requests between browsers and APIs while relaxing the restrictive same-origin policy. 
- It prevents unauthorized access to resources across different origins through header-based validation.
- Proper implementation blocks malicious sites from exploiting legitimate APIs.

- **Origin Definition** -
  - An origin combines scheme (protocol), hostname, and port number. 
  - Requests match origins only if all three components are identical (e.g., https://api.example.com:443 differs from http://api.example.com). 
  - Cross-origin requests from differing origins trigger CORS checks.

- **Same-Origin Policy (SOP)** -
  - SOP blocks cross-origin resource access by default, protecting against attacks like CSRF where malicious sites (e.g., evil-site.com) access sensitive data from trusted domains (e.g., facebook.com).
  - SOP permits only same-origin interactions, such as a site fetching its own images.

- **Simple Requests** -
  - Simple requests bypass preflight if they use safe methods (`GET`, `HEAD`, `POST`) with standard headers and Content-Type values (`application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`).

- Client sends `Origin: http://abc.com`; server responds `Access-Control-Allow-Origin: http://abc.com` to approve. 
- Mismatched origins trigger browser rejection with minimal error details.

- **Preflight Requests** -
  - Non-simple requests (custom headers, unsafe methods like `PUT`/`DELETE`, or non-standard `Content-Type`) require O`PTIONS preflight.
  - Browser sends -
  ```
  OPTIONS /resource HTTP/1.1
  Origin: http://abc.com
  Access-Control-Request-Method: POST
  ```

  - Server approves via `Access-Control-Allow-Methods: POST` and `Access-Control-Max-Age` for caching.
  - Credentials require `withCredentials=true` client-side and `Access-Control-Allow-Credentials: true` server-side.

- **Key Vulnerabilities** -
  - Wildcard origins (`Access-Control-Allow-Origin: *`) expose APIs to any domain.
  - Insufficient origin validation permits malicious subdomains.
  - Trusted origins vulnerable to XSS enable CORS abuse.
  - Missing server-side checks bypass CORS entirely.
​
- **Prevention Best Practices** -
  - Validate origins against explicit allowlist; reject null/empty origins.
  - Avoid wildcards; use specific domains only.
  - Implement short Access-Control-Max-Age timeouts.
  - Combine with server-side authentication and input validation.
  - For credentials, never use wildcard origins.
​
​## Authentication vs Authorization

- Authentication verifies a user's identity, while authorization determines their access permissions. 
- **Core Concepts** -
  - APIs require authentication to confirm "who you are," often via credentials like usernames/passwords or tokens. 
  - Authorization follows to check "what you can do," such as read versus edit permissions, using models like RBAC or scopes. 
  - In a banking API, authentication prevents imposters from logging in, while authorization limits transaction approvals to verified accounts.

- **HTTP Basic Authentication** -
  - This method sends Base64-encoded username:password in the Authorization header after a 401 challenge with WWW-Authenticate. 
  - The realm keyword in WWW-Authenticate specifies the protected scope, like "api.example.com".

- **API Keys** -
  - Unique strings generated per application, sent in headers (e.g., `Authorization: Bearer <key>`). 
  - Servers validate against a database; revocable and refreshable, but identify apps, not end-users.

- **JSON Web Tokens (JWT)** -
  - Compact tokens in header.payload.signature format, Base64-encoded and signed. 
  - Payload holds claims (e.g., user ID, expiry); stateless verification without database lookups.

​### Comparison

| Aspect          | Advantages                        | Disadvantages                                                                                 |
| --------------- | --------------------------------- | --------------------------------------------------------------------------------------------- |
| HTTP Basic Auth | Simple, fast, native HTTP support | Passwords sent per request; only encoded (needs HTTPS); no sessions or revocation |
| API Keys | Simple; multiple per account; revocable; isolates breaches | No user auth; interception risk; needs extra auth for permissions​ |
| JWT    | Scalable; self-contained authZ; temporary access | Overhead parsing/signing; revocation tricky; crypto knowledge needed​ |

