# Remote Procedure Calls (RPCs)

- RPC provide a function-call-like abstraction for executing code across distributed systems and microservices.
- Introduced in the late 1970s, RPC standardizes interprocess communication while hiding network complexities.

## RPC Architecture Components

| Component             | Description                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| Client Stub           | Local proxy that marshals function parameters into RPC messages and forwards to runtime. educative​     |
| Server Stub           | Remote proxy that unmarshals messages and invokes actual server functions. educative​                   |
| Runtime RPC           | Core engine handling network transport, connection management, and error recovery. educative​           |
| IDL Compiler          | Generates language-specific stubs from Interface Definition Language (IDL) specifications. educative​   |
| Import/Export Modules | Client-side lists available remote functions; server-side registers implementable functions. educative​ |

## Modern RPC Implementations

- **gRPC** - High-performance RPC using Protocol Buffers (protobuf) serialization over HTTP/2.
- **Thrift** - Facebook's cross-language RPC framework with IDL-based code generation.
- **RMI** - Java-specific object serialization over RPC for JVM-to-JVM communication.
- **SOAP** - XML/JSON-based RPC protocol (legacy, being replaced by gRPC).
​
## RPC Workflow (Request-Response)

- **Client Call** - Application invokes stub → marshals parameters → sends to RPC runtime.
- **Network Binding** - Static (known endpoint) or dynamic (DNS-resolved) connection setup.
- **Transmission** - Runtime uses TCP/UDP (typically HTTP) → server stub unmarshals → executes function.
- **Response** - Server marshals result → transmits back → client stub unmarshals → returns value.
​
## RPC Message Format

- JSON-RPC Example -
```
Request:
POST /rpc-demo HTTP/1.1
{
  "jsonrpc": "2.0",
  "method": "demo", 
  "params": {"greeting": "Hi"},
  "id": "99"
}

Response:
HTTP/1.1 200 OK
{
  "jsonrpc": "2.0",
  "id": "99",
  "result": "Hello!"
}
```

## Pros & Cons

- **Pros** -
  - Simple function-call semantics hide network complexity.
  - IDL enables cross-language code generation and type safety.
  - High performance (gRPC over HTTP/2 with binary protobuf).
  - Scales well for microservices communication.
​
- **Cons** -
  - Tight Coupling - Client/server changes require coordinated updates.
  - Synchronous by Default - Blocks caller until response (though async variants exist).
  - Language Limitations - May need custom stubs for unsupported languages.
