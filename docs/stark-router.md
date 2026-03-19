# Stark router

Stark is the core class of the framework. It manages route registration, request dispatch, and OpenAPI spec generation.

## Fields

| Field        | Access  | Description                                                  |
|--------------|---------|--------------------------------------------------------------|
| `Handlers`   | Public  | Namespace (or class instance) where handler functions live   |
| `ThreadMode` | Public  | Controls Jarvis threading -- `''`, `0`, `1`, `'DEBUG'`, `'AUTO'` |
| `Info`       | Public  | Namespace with `title`, `version`, and optional `description` for the OpenAPI info block |
| `Debug`      | Public  | Bitmask controlling debug stops and logging (see [Debug mode](#debug-mode)) |
| `OnErrorFn`  | Public  | Name of a dyadic result-returning function in `Handlers` to call on handler errors (see [Error handling](#error-handling)) |

## Route registration

### Single-route methods

Register one route at a time by calling the HTTP verb method dyadically: path on the left, handler spec on the right.

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

### Bulk registration

Register multiple routes at once with `router.Register`. There are two input modes: a matrix or a vector of namespaces.

#### Matrix mode

Pass a matrix where each row is one route:

```apl
routes←[
    ⍝ Method    Endpoint                                Handler        Schema
      'GET'     '/'                                     'Root'         RootSchema
      'GET'     '/items'                                'ListItems'    ListItemsSchema
      'GET'     '/items/{id}'                           'GetItem'      GetItemSchema
      'POST'    '/items'                                'CreateItem'   CreateItemSchema
      'PUT'     '/items/{id}'                           'UpdateItem'   UpdateItemSchema
      'DELETE'  '/items/{id}'                           'DeleteItem'   DeleteItemSchema
      'GET'     '/customer/{cust_id}/invoice/{inv_id}'  'GetInvoice'   GetInvoiceSchema
]
router.Register routes
```

The schema column is optional -- omit it for a 3-column matrix and Stark fills in empty options:

```apl
routes←[
    'GET'  '/items'       'ListItems'
    'POST' '/items'       'CreateItem'
    'GET'  '/items/{id}'  'GetItem'
]
router.Register routes
```

#### Namespace vector mode

Pass a vector of namespaces, each with `.method`, `.path`, `.handler`, and optionally `.spec`:

```apl
router.Register (
    (method: 'GET'  ⋄ path: '/items'      ⋄ handler: 'ListItems')
    (method: 'POST' ⋄ path: '/items'      ⋄ handler: 'CreateItem')
    (method: 'GET'  ⋄ path: '/items/{id}' ⋄ handler: 'GetItem' ⋄ spec: GetItemSchema)
)
```

The `.spec` field is equivalent to the schema column in matrix mode -- a namespace with metadata for [OpenAPI generation](openapi.md). Omit it for routes that need no metadata.

Namespace mode routes go through the same duplicate-detection logic as single-route methods (`Get`, `Post`, etc.).

#### Mixing registration styles

Both `Register` and the individual verb methods can be used together:

```apl
'/health' router.Get 'Health'        ⍝ registered individually
router.Register routes               ⍝ bulk-register the rest
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

## Error handling

By default, any error raised inside a handler is re-signaled out of Stark's dispatch loop, propagating to the underlying Jarvis server (which returns a 500 with no body).

Set `OnErrorFn` to the name of a handler function in your `Handlers` namespace to intercept errors instead:

```apl
router.OnErrorFn←'HandleError'
```

When an error occurs and `OnErrorFn` is set, Stark:

1. Clones `⎕DMX` into a plain namespace before anything else can change it.
2. Calls `req.Fail 500` to set the response status.
3. Calls `result←err(Handlers⍎OnErrorFn)req` and uses the return value as the response body.

The hook receives the cloned `⎕DMX` as its left argument and `req` as its right argument:

```apl
∇ result←err HandleError req
  ⍝ err.EN      - error number
  ⍝ err.EM      - error message
  ⍝ err.Message - full error text
  result←(error: err.Message)
∇
```

The function must be result-returning and dyadic (or ambivalent). Stark validates this at `Start` and signals EN 11 if the requirement is not met.

!!! note
    `OnErrorFn` is bypassed when `Debug←1` is set, because `Debug←1` disables the `:Trap` block entirely. This is intentional: debug mode lets errors propagate to the APL session for inspection.

## Inspection

```apl
router.Routes        ⍝ returns a matrix of route descriptors (columns: method, path, handler, options)
router.OpenAPI       ⍝ returns the generated OpenAPI spec as a namespace
```

`Routes` returns an `N×4` matrix. Each row is one user-registered route (internal routes such as `/openapi.json` are excluded). The columns are:

| Column | Content   | Example           |
|--------|-----------|-------------------|
| 1      | Method    | `'GET'`           |
| 2      | Path      | `'/items/{id}'`   |
| 3      | Handler   | `'GetItem'`       |
| 4      | Options   | metadata namespace or `()` |

```apl
⍝ Count of registered routes
≢router.Routes

⍝ All methods
router.Routes[;1]

⍝ Handler for the first route
3⊃router.Routes[1;]
```
