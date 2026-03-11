# Jarvis server

Jarvis is a Dyalog APL HTTP/HTTPS server library that Stark uses under the hood. You generally don't interact with Jarvis directly when using Stark, but understanding it can help with advanced configuration.

## Role in Stark

When you call `router.Start port`, Stark creates a Jarvis instance configured in REST mode and wires all incoming requests through its dispatcher. The Jarvis instance handles:

- TCP/IP networking via the Conga library
- HTTP protocol parsing (headers, body, query strings)
- Response serialization to JSON
- Thread management
- CORS headers
- Gzip/deflate compression
- Session management
- HTTPS/TLS (when configured)

## Key Jarvis settings

These are set internally by Stark but are worth knowing:

| Setting                | Value set by Stark | Purpose                                |
|------------------------|-------------------|----------------------------------------|
| `Paradigm`             | `'REST'`          | REST mode dispatches by HTTP method    |
| `CodeLocation`         | Internal namespace | Where Jarvis looks for verb handlers   |
| `RESTFailProcessing`   | `1`               | Allows custom error responses          |
| `Port`                 | From `Start` arg  | Listening port                         |
| `DYALOG_JARVIS_THREAD` | From `ThreadMode` | Thread handling strategy               |
| `Debug`                | Derived from `router.Debug` | Stark forwards bits `1`, `4`, `8`, and `16`. Bits `2`, `32`, and `64` are Stark-specific and handled internally. |

## Direct Jarvis access

If you need the underlying Jarvis instance (for example, to configure HTTPS or CORS), use `GetStark` from your app class to get the router, then access the Jarvis instance through the class internals, or configure Jarvis settings before calling `Start`.

## Further reading

Jarvis is maintained by Dyalog Ltd. For full documentation on Jarvis features like HTTPS configuration, authentication, and session management, refer to the [Dyalog Jarvis repository](https://github.com/Dyalog/Jarvis).
