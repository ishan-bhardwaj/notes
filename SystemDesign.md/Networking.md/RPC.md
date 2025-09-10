# Remote Procedure Calls (RPCs)

- A protocol for interprocess communication spanning transport and application layers - allowing a program on one machine to execute procedures on another machine as if they were local calls.
- Components (5): Client, Client Stub, Server, Server Stub, RPC Runtime.
    - Client, Client Stub, RPC Runtime → client machine.
    - Server, Server Stub, RPC Runtime → server machine.
- The client, the client stub, and one instance of RPC runtime runs on the client machine. The server, the server stub, and one instance of RPC runtime runs on the server machine.
- Steps during RPC process -
    - Client initiates a client stub process (stored in the address space of the client) and provide parameters to it.
    - The client stub converts the parameters into a standardized format and packs them into a message. The client stub requests the local RPC runtime to deliver the message to the server.
    - The RPC runtime at the client delivers the message to the server over the network and then waits for the message result from the server.
    - RPC runtime at the server receives the message and passes it to the server stub.
    - The server stub unpacks the message, takes the parameters out of it, and calls the desired server routine, using a local procedure call.
    - The result is returned to the server stub. The server stub packs the returned result into a message and sends it to the RPC runtime at the server on the transport layer.
    - The server’s RPC runtime returns the packed result to the client’s RPC runtime over the network.
    - The client’s RPC runtime that was waiting for the result now receives the result and sends it to the client stub.
    - The client stub unpacks the result, and the execution process returns to the caller at this point.

> [!NOTE]
> The RPC runtime is responsible for transmitting messages between client and server via the network. The responsibilities of RPC runtime also include retransmission, acknowledgment, and encryption.


