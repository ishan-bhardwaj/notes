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
## Authentication vs Authorization

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

### Comparison

| Aspect          | Advantages                        | Disadvantages                                                                                 |
| --------------- | --------------------------------- | --------------------------------------------------------------------------------------------- |
| HTTP Basic Auth | Simple, fast, native HTTP support | Passwords sent per request; only encoded (needs HTTPS); no sessions or revocation |
| API Keys | Simple; multiple per account; revocable; isolates breaches | No user auth; interception risk; needs extra auth for permissions​ |
| JWT    | Scalable; self-contained authZ; temporary access | Overhead parsing/signing; revocation tricky; crypto knowledge needed​ |

## OAuth

- OAuth 2.0 provides a standardized framework for token-based authorization, allowing secure delegated access to user resources without sharing credentials. 
- Developed in 2010 building on OAuth 1.0 from 2007, it addresses limitations of basic protocols like API keys and JWTs by enabling safe token exchange.

- OAuth involves four core actors coordinating secure access -
  - Resource Owner (end-user) owns protected data.
  - Resource Server (API) hosts it.
  - Client (app like Spotify) requests access.
  - Authorization Server issues tokens after user consent.

- OAuth supports multiple grant types (flows) tailored to scenarios -

| Flow               | User Permission | Access User Data | Refreshable | Best For                                                                              |
| ------------------ | --------------- | ---------------- | ----------- | ------------------------------------------------------------------------------------- |
| Authorization Code | Yes             | Yes              | Yes         | Server-side web appsauth0​                                                            |
| Auth Code + PKCE   | Yes             | Yes              | Yes         | SPAs/mobile (prevents code interception via code_verifier/challenge)developer.okta+1​ |
| Client Credentials | No              | No               | No          | Machine-to-machine, public resourceseducative​                                        |
| Implicit           | Yes             | Yes              | No          | Legacy browsers (deprecated due to token exposure in redirects)stackoverflow+1​       |

- **Authorization Code Flow** -
  - Client redirects user to Authorization Server with client_id, redirect_uri, scope; user consents, gets code; client exchanges code for access token via backend POST (secure from interception). 
  - Resource Server validates token signature for data access.
​
- **PKCE Extension** -
  - Enhances code flow for public clients.
  - Client generates code_verifier, derives code_challenge (SHA256 hash), sends challenge initially; later sends verifier—server matches hashes to prove legitimacy, blocking attacker code swaps.
​
- **Client Credentials Flow** -
  - Client authenticates directly with Authorization Server using its credentials (no user involved), receives token for its own resources like public APIs.
​
- **Implicit Flow Risks** -
  - Directly returns access token in redirect URI (no code exchange), exposing it in browser history/URL—avoid; use PKCE instead.

## Authentication and Authorization Frameworks: OpenID and SAML

- OpenID Connect (OIDC) and SAML provide authentication frameworks that complement OAuth 2.0's authorization capabilities by verifying user identity. 
- OIDC extends OAuth with ID tokens for SSO across apps, while SAML uses XML assertions for enterprise SSO with both authn and authz.

- **Core Differences** -
  - OAuth 2.0 authorizes resource access via tokens but lacks user identity verification—like a movie ticket without owner details. 
  - OIDC adds authentication atop OAuth flows, delivering ID tokens with user claims (sub, name, email). 
  - SAML independently handles both via XML-based trust between Identity Providers (IdP) and Service Providers (SP).

- **OpenID Connect Process** -
  - OIDC follows OAuth flows (typically Authorization Code + PKCE) but requests scope=openid to get an ID token alongside access tokens. 
  - Client redirects user to Authorization Server; after consent, server returns ID token (JWT) to redirect_uri. 
  - Client validates signature, nonce, expiry, then uses claims for session and access token for APIs.
    - ID Token vs Access Toke -: ID tokens authenticate identity (user-facing, contains claims); access tokens authorize API calls (opaque or JWT, resource-specific).
​
| Aspect | Advantages                                               | Disadvantages                                                       |
| ------ | -------------------------------------------------------- | ------------------------------------------------------------------- |
| OIDC   | JSON/RESTful; mobile/SPA-friendly; SSO; pairs with OAuth | Relies on OAuth; phishing risks; less enterprise adoptioneducative​ |

- **SAML 2.0 Process** -
  - SAML establishes trust between IdP and SP. 
  - User accesses SP, gets redirected to IdP for authn; IdP issues SAML assertion (XML with authn statement, attributes, signature). 
  - User posts assertion to SP, which validates against IdP metadata and grants access. 
  - Supports SSO across federated domains. 
    - Trust Setup - IdPs/SPs exchange metadata (certificates, endpoints) via out-of-band config for signature validation.

| Aspect | Advantages                                                  | Disadvantages                                                     |
| ------ | ----------------------------------------------------------- | ----------------------------------------------------------------- |
| SAML   | Enterprise-grade; built-in authz; no password storage at SP | XML overhead; complex; weak for SPAs/mobile; MITM riskseducative​ |

- **Framework Comparison** -

| Framework | Focus                   | Format               | Best For              | Drawbacks                   |
| --------- | ----------------------- | -------------------- | --------------------- | --------------------------- |
| OAuth 2.0 | Authorization           | JSON tokens          | APIs/mobile           | No identity proofeducative​ |
| OIDC      | Authentication (+OAuth) | JSON (JWT ID tokens) | Modern web/mobile SSO | Needs OAuth base            |
| SAML      | Authn + Authz           | XML assertions       | Enterprise B2B/B2E    | Heavyweight; legacy UI      |

> [!TIP]
> Use OIDC+OAuth for consumer apps; SAML for enterprises needing strong federation.

## Zero Trust Model

- Zero Trust assumes breach - never trust networks, verify explicitly every request (identity, context), and limit privileges to essentials. 
- For APIs, enforce continuous authn (OAuth/OIDC/MFA), real-time monitoring, schema validation, and end-to-end encryption per call. 
- Reshapes authz by denying implicit trust—internal microservices get mutual TLS; external users face adaptive risk scoring.
​- **Key Practices** - Behavior analytics detect anomalies; micro-segmentation isolates resources; no perimeter reliance.

## WAAP Overview

- Web Application and API Protection (WAAP) integrates WAF, DDoS mitigation, bot management, and API-specific defenses like schema validation and rate limiting. 
- Uses AI for real-time anomaly detection, payload inspection, and OAuth/JWT enforcement against injections/exfiltration. 
- Essential for public/partner APIs facing volumetric attacks or credential stuffing—beyond traditional WAF for cloud-native scale.

| WAAP Components          | Protects Against            | Benefits                    |
| ------------------------ | --------------------------- | --------------------------- |
| WAF + API Inspection     | Injections, schema abuse    | Fine-grained policyradware​ |
| DDoS/Bot Mitigation      | Volumetric floods, scraping | Availability assurance      |
| Rate Limiting/Anomaly AI | Abuse, unknown threats      | Scalable enforcementkmcd​   |

