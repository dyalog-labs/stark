# Jarvis server

Jarvis is a Dyalog APL HTTP/HTTPS server library that Stark uses under the hood. You generally don't interact with Jarvis directly when using Stark, but understanding it can help with advanced configuration.

## Role in Stark

Each Stark router creates its own Jarvis instance. When you call `router.Start port`, Stark configures that instance in REST mode and wires all incoming requests through its dispatcher. The Jarvis instance handles:

- TCP/IP networking via the Conga library
- HTTP protocol parsing (headers, body, query strings)
- Response serialization to JSON
- Thread management
- CORS headers
- Gzip/deflate compression
- Session management (when configured)
- HTTPS/TLS (when configured)

## Key Jarvis settings

`Start` sets these on the Jarvis instance every time it is called:

| Setting                | Value set by Stark | Purpose                                |
|------------------------|-------------------|----------------------------------------|
| `Paradigm`             | `'REST'`          | REST mode dispatches by HTTP method    |
| `CodeLocation`         | Internal namespace | Where Jarvis looks for verb handlers   |
| `RESTFailProcessing`   | `1`               | Allows custom error responses          |
| `Port`                 | From `Start` arg  | Listening port                         |
| `DYALOG_JARVIS_THREAD` | From `ThreadMode` | Thread handling strategy (only set if `ThreadMode` is not `''`) |
| `Debug`                | Derived from `router.Debug` | Stark forwards bits `1`, `4`, `8`, and `16`. Bits `2`, `32`, and `64` are Stark-specific and handled internally. |

## Direct Jarvis access

To configure anything else on the underlying Jarvis instance, such as HTTPS or CORS, use the `GetJarvis` method on the Stark instance before calling `Start`:

```apl
j←router.GetJarvis
j.CORS_Origin←'https://example.com'   ⍝ restrict CORS to one origin (default '*')
router.Start 8080
```

Settings in the table above are overwritten by `Start`, so set them through Stark instead (`ThreadMode`, `Debug`, and the `Start` argument).

## Further reading

Jarvis is maintained by Dyalog Ltd. For full documentation on Jarvis features like HTTPS configuration, authentication, and session management, refer to the [Dyalog Jarvis repository](https://github.com/Dyalog/Jarvis) or the [Jarvis documentation](https://dyalog.github.io/Jarvis/latest/).
