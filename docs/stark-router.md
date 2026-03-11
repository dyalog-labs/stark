# Stark router

Stark is the core class of the framework. It manages route registration, request dispatch, and OpenAPI spec generation.

## Fields

| Field        | Access  | Description                                                  |
|--------------|---------|--------------------------------------------------------------|
| `Handlers`   | Public  | Namespace (or class instance) where handler functions live   |
| `ThreadMode` | Public  | Controls Jarvis threading -- `''`, `0`, `1`, `'DEBUG'`, `'AUTO'` |
| `Info`       | Public  | Namespace with `title`, `version`, and optional `description` for the OpenAPI info block |
| `Debug`      | Public  | Bitmask controlling debug stops and logging (see [Debug mode](#debug-mode)) |

## Route registration

Register routes by calling the HTTP verb method dyadically: path on the left, handler spec on the right.

```apl
path router.Get    rarg
path router.Post   rarg
path router.Put    rarg
path router.Delete rarg
path router.Patch  rarg
```

`rarg` is either:

- A simple string -- the handler function name:
  ```apl
  '/items' router.Get 'ListItems'
  ```
- A two-element vector -- handler name and a metadata namespace:
  ```apl
  '/items' router.Get ('ListItems' schema)
  ```

## Path parameters

Use curly braces to define dynamic segments in a route pattern:

```apl
'/users/{id}'                          router.Get 'GetUser'
'/customer/{cust_id}/invoice/{inv_id}' router.Get 'GetInvoice'
```

Inside the handler, path parameter values are available on `req.PathParams`:

```apl
∇ result←GetUser req
  id←req.PathParams.id       ⍝ string value from the URL
∇
```

Multiple parameters work the same way:

```apl
∇ result←GetInvoice req
  cust←req.PathParams.cust_id
  inv←req.PathParams.inv_id
∇
```

!!! note
    Path parameter values are always strings. Use `⎕VFI` to convert to numbers when needed:
    ```apl
    id←⊃2⊃⎕VFI req.PathParams.id
    ```

## Query parameters

Query string parameters are parsed into `req.QueryParams` as a namespace:

```apl
⍝ GET /search?q=dyalog
∇ result←Search req
  q←req.QueryParams ⎕VGET ⊂(,'q') ''   ⍝ default to '' if missing
∇
```

## Request object

Handlers receive a Jarvis request object (`req`) with these key members:

| Member         | Description                              |
|----------------|------------------------------------------|
| `Method`       | HTTP method (`'GET'`, `'POST'`, etc.)    |
| `Endpoint`     | Request path                             |
| `Payload`      | Parsed JSON body (for POST/PUT/PATCH)    |
| `PathParams`   | Namespace of path parameter values       |
| `QueryParams`  | Namespace of query parameter values      |
| `StatusCode`   | Set this to override the response status |
| `Fail code`    | Call to return an error status code      |

## Handler functions

A handler is a monadic function that takes `req` and returns a namespace (which Jarvis serializes to JSON).

```apl
∇ result←MyHandler req
  :Access Public
  result←(key: 'value')
∇
```

To return a non-200 status:

```apl
⍝ Set a custom success status
req.StatusCode←201
result←item

⍝ Return an error
req.Fail 404
result←(detail: 'Not found')
```

## Route metadata

Metadata is a namespace passed alongside the handler name. It drives the [OpenAPI spec](openapi.md).

```apl
schema←(
    summary: 'List all items'
    description: 'Returns every item in the database'
    tags: ('items'⋄)
    body: (type: 'object' ⋄ properties: (name: (type: 'string')))
    response: (201 (type: 'object' ⋄ properties: (id: (type: 'integer'))))
    errors: ((404 'Not found')⋄)
)
'/items' router.Post ('CreateItem' schema)
```

| Key           | Type                          | Description                            |
|---------------|-------------------------------|----------------------------------------|
| `summary`     | String                        | Short description of the operation     |
| `description` | String                        | Longer description                     |
| `tags`        | Vector of strings             | OpenAPI tags for grouping              |
| `body`        | Schema namespace              | Request body JSON Schema               |
| `response`    | `(statusCode schema)`         | Success response code and schema       |
| `errors`      | Vector of `(code description)` | Error response descriptions            |

## Debug mode

Set `router.Debug` before calling `Start` to enable debug stops and logging. The value is a bitmask — combine levels by adding their values.

```apl
router.Debug←2      ⍝ stop before each user handler call
router.Debug←1+2    ⍝ stop on error AND stop before handler
```

| Value | Meaning |
|-------|---------|
| `1`   | **Stop on error** — disables error traps at both the Stark dispatch level and the underlying Jarvis level, so errors in user handlers propagate all the way to the APL session. Forwarded to Jarvis. |
| `2`   | **Stop before user handler** — execution stops just before your handler function is called, letting you inspect `req`, `req.PathParams`, `req.QueryParams`, etc. |
| `4`   | Jarvis framework debugging (forwarded to Jarvis). |
| `8`   | Conga event logging — logs low-level TCP/IP events (forwarded to Jarvis). |
| `16`  | Stop just before the HTTP response is sent (forwarded to Jarvis). |
| `32`  | **Stark framework debug** — prints a trace line to the session for each matched user route: `STARK: GET /users/42 → GetUser` |
| `64`  | **Stop before routing** — execution stops at the entry of `_Dispatch`, before any route matching. Useful for inspecting the raw `req` object as Stark sees it. |

Bits `2`, `32`, and `64` are handled entirely by Stark. Bits `1`, `4`, `8`, and `16` are forwarded to the underlying Jarvis instance.

!!! note
    `Debug←1` must disable traps at every level in the call stack — Stark's dispatch, Jarvis's request handler, and Jarvis's server loop — for errors to reach the APL session. This is why bit `1` is forwarded to Jarvis. Whether the session actually stops interactively depends on your thread mode; it works most naturally with `ThreadMode←'DEBUG'` or when running in thread 0.

## Lifecycle methods

```apl
router.Start 8080    ⍝ start serving on the given port
router.Stop          ⍝ shut down the server
```

## Inspection

```apl
router.Routes        ⍝ returns the internal vector of route descriptors
router.OpenAPI       ⍝ returns the generated OpenAPI spec as a namespace
```
