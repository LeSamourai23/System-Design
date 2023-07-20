Server-Sent Events (SSE) is a way of establishing long-term communication between client and server that enables the server to proactively push data to the client.

![[server-sent-events.webp]]

It is unidirectional, meaning once the client sends the request it can only receive the responses without the ability to send new requests over the same connection.

### Working

1. The client makes a request to the server.
2. The connection between client and server is established and it remains open.
3. The server sends responses or events to the client when new data is available.

### Advantages

- Simple to implement and use for both client and server.
- Supported by most browsers.
- No trouble with firewalls.

### Disadvantages

- Unidirectional nature can be limiting.
- Limitation for the maximum number of open connections.
- Does not support binary data.