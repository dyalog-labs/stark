# Stark router

Stark is the core class of the framework. It manages route registration, request dispatch, and OpenAPI spec generation.

## Creating a router

```apl
router←Stark.New ()
```

`Stark.New` returns a new router whose `Handlers` is the namespace (or class instance) that called it. The router registers the internal `/openapi.json` route as soon as it is created.

## Fields

| Field        | Default                              | Description                                                  |
|--------------|--------------------------------------|--------------------------------------------------------------|
| `Handlers`   | The caller of `Stark.New`            | Namespace (or class instance) where handler functions live   |
| `ThreadMode` | `''`                                 | How the Jarvis server runs relative to your session (see [Thread mode](#thread-mode)) |
| `Info`       | `(title: 'AStarkAPI' ⋄ version: '1.0.0')` | Namespace with `title`, `version`, and optional `description` for the OpenAPI `info` block |
| `Spec`       | `()`                                 | Namespace of additional root-level OpenAPI fields (`components`, `security`, `servers`, etc.); merged into the spec alongside `info` |
| `Debug`      | `0`                                  | Bitmask controlling debug stops and logging (see [Debug mode](#debug-mode)) |
| `OnErrorFn`  | `''`                                 | Name of a dyadic result-returning function in `Handlers` to call on handler errors (see [Error handling](#error-handling)) |

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

The schema column is optional -- omit it for a 3-column matrix:

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

`Register` is all-or-nothing: if any route is rejected, none of the routes passed in that call are registered. Its shy result is the same as [`router.Routes`](#inspection).

#### Mixing registration styles

Both `Register` and the individual verb methods can be used together:

```apl
'/health' router.Get 'Health'        ⍝ registered individually
router.Register routes               ⍝ bulk-register the rest
```

### Duplicate routes

Every registration method rejects a route that duplicates an existing one, signalling EN 11. Two routes are duplicates when they have the same method (compared case-insensitively) and the same path structure. Parameter names and trailing slashes are ignored, so these all conflict with `GET /items/{id}`:

```apl
'/items/{item_id}' router.Get 'GetItem'
'/items/{id}/'     router.Get 'GetItem'
```

The same path with a different method is not a duplicate.

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

Parameter names that are not valid APL names are [mangled with `7162⌶`](#mangled-names).

## Route matching

- Literal path segments are matched case-sensitively; the HTTP method is matched case-insensitively.
- Empty segments are ignored, so `/items/`, `//items` and `/items` are the same path.
- When a literal segment and a `{param}` segment could both match, the literal wins. With both `/items/special` and `/items/{id}` registered, `GET /items/special` goes to the first and `GET /items/42` to the second.
- A request that matches no route returns `404` with the body `{"detail":"Not Found"}`. This includes a known path requested with an unregistered method (there is no `405`).

## Query parameters

Query string parameters are parsed into `req.QueryParams` as a namespace. Values are strings:

```apl
⍝ GET /search?q=dyalog
∇ result←Search req
  q←req.QueryParams ⎕VGET ⊂(,'q') ''   ⍝ default to '' if missing
∇
```

Parameter names that are not valid APL names are [mangled with `7162⌶`](#mangled-names).

## Request object

Handlers receive a Jarvis request object (`req`) with these key members:

| Member             | Description                                  |
|--------------------|----------------------------------------------|
| `Method`           | HTTP method (`'GET'`, `'POST'`, etc.)        |
| `Endpoint`         | Request path                                 |
| `Payload`          | Parsed JSON body (for POST/PUT/PATCH)        |
| `Body`             | Raw request body                             |
| `Headers`          | Matrix of request header names and values    |
| `GetHeader name`   | Returns the value of a request header        |
| `PathParams`       | Namespace of path parameter values           |
| `QueryParams`      | Namespace of query parameter values          |
| `SetStatus code`   | Sets the response status code                |
| `Fail code`        | Sets an error status code                    |

See the [Jarvis documentation](https://dyalog.github.io/Jarvis/latest/) for the full request object.

## Handler functions

A handler is a monadic, result-returning function that takes `req` and returns the response body, usually a namespace, which Jarvis serializes to JSON.

```apl
∇ result←MyHandler req
  :Access Public
  result←(key: 'value')
∇
```

In a class, handlers must be `:Access Public` so that Stark can call them.

Handler names must not begin with `_`. Stark reserves those names for its internal handlers (such as the one serving `/openapi.json`); it runs them itself rather than looking them up in `Handlers`, and leaves them out of the OpenAPI spec and `router.Routes`.

To return a non-200 status:

```apl
⍝ Set a custom success status
req.SetStatus 201
result←item

⍝ Return an error
req.Fail 404
result←(detail: 'Not found')
```

## Route metadata

Metadata is a namespace passed alongside the handler name. It drives the [OpenAPI spec](openapi.md).

```apl
schema←(
    summary: 'Create a new item'
    description: 'Creates an item and returns it'
    tags: ('items'⋄)
    requestBody: (
        required: ⊂'true'
        content: (⍙application⍙47⍙json: (schema: (type: 'object' ⋄ properties: (name: (type: 'string')))))
    )
    responses: (201 (type: 'object' ⋄ properties: (id: (type: 'integer')))⋄)
)
'/items' router.Post ('CreateItem' schema)
```

The metadata namespace is merged directly into the OpenAPI operation object — any valid [OpenAPI operation field](https://spec.openapis.org/oas/v3.0.3#operation-object) passes through untouched. Use OpenAPI field names directly: `requestBody`, `parameters`, `security`, `deprecated`, etc.

Stark applies a small number of conveniences automatically, such as the `responses` shorthand used above — see [OpenAPI generation](openapi.md#conveniences) for the full list.

### Mangled names

APL names cannot contain characters such as `/` or `-`, so keys like these are written in the form produced by `7162⌶`. For example, `application/json` becomes `⍙application⍙47⍙json`:

```apl
      0(7162⌶)'application/json'
⍙application⍙47⍙json
```

The same applies to incoming path and query parameter names: `?page-size=10` arrives as `req.QueryParams.⍙page⍙45⍙size`.

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

## Thread mode

`ThreadMode` sets Jarvis's `DYALOG_JARVIS_THREAD`, which decides where the server loop runs. Set it before calling `Start`:

| Value     | Behaviour |
|-----------|-----------|
| `''`      | Same as `'AUTO'` (the default) |
| `'AUTO'`  | `'DEBUG'` in an interactive session, otherwise `1` |
| `'DEBUG'` | Server runs in a background thread; `Start` returns and the session stays usable |
| `0`       | Server runs in thread 0; `Start` does not return until the server stops |
| `1`       | Server runs in a separate thread, but `Start` waits on it (`⎕TSYNC`) and does not return until the server stops |

```apl
router.ThreadMode←'DEBUG'
router.Start 8080
```

Use `'DEBUG'` for interactive development and `0` or `1` for scripts and services, where the process should stay alive while the server runs.

## Lifecycle methods

```apl
router.Start 8080    ⍝ start serving on the given port
router.Stop          ⍝ shut down the server
router.GetJarvis     ⍝ the underlying Jarvis instance (see Jarvis server)
```

See [Jarvis server](jarvis-server.md#direct-jarvis-access) for what you can configure through `GetJarvis`.

## Error handling

By default, any error raised inside a handler is re-signaled out of Stark's dispatch loop, propagating to the underlying Jarvis server, which returns a 500 with no body.

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

The function must be result-returning and dyadic (or ambivalent). Stark validates this at `Start` and signals EN 11 if the requirement is not met. In a class, it must be `:Access Public`, like any other handler.

!!! note
    `OnErrorFn` is bypassed when `Debug←1` is set, because `Debug←1` disables the `:Trap` block entirely. This is intentional: debug mode lets errors propagate to the APL session for inspection.

## Inspection

```apl
router.Routes        ⍝ returns a matrix of route descriptors (columns: method, path, handler, operation)
router.OpenAPI       ⍝ returns the generated OpenAPI spec as a namespace
```

`Routes` returns an `N×4` matrix. Each row is one user-registered route (internal routes such as `/openapi.json` are excluded). The columns are:

| Column | Content   | Example           |
|--------|-----------|-------------------|
| 1      | Method    | `'GET'`           |
| 2      | Path      | `'/items/{id}'`   |
| 3      | Handler   | `'GetItem'`       |
| 4      | Operation | pre-built OpenAPI operation namespace |

```apl
⍝ Count of registered routes
≢router.Routes

⍝ All methods
router.Routes[;1]

⍝ Handler for the first route
3⊃router.Routes[1;]
```
