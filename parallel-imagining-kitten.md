# Jarvis Routing System — Design Document

## Context

Dyalog APL's Jarvis HTTP server provides raw REST endpoint handling, but has no built-in routing, schema metadata, or OpenAPI support. The existing `FastJarvis.aplc` prototype proves the concept — a class that wraps Jarvis and dispatches requests based on method + path pattern matching — but its route registration is minimal (`'METHOD /path' 'handlerName'` pairs) with no place to attach metadata.

This plan designs a routing system based on **Idea B** from Ideas.md: Fastify-style registration where each route call accepts an optional APLAN options namespace carrying metadata (schemas, summaries, tags, etc.). The core deliverable is routing and dispatch. The design must be **extensible** so that validation, OpenAPI generation, middleware, and error handling can be added later without changing the registration API.

---

## Design Principles

1. **Progressive enhancement** — `router.Post '/users' 'CreateUser'` must work with zero metadata. Adding an APLAN options argument enriches the route without changing the handler.
2. **Routes as data** — Every registered route is an inspectable namespace. Future features (OpenAPI, validation) walk the route registry; they don't require changes to how routes are registered.
3. **Handlers are plain functions** — A handler takes a request namespace, returns a result namespace. The router never forces handlers to inherit from a class or follow a special protocol.
4. **Built on Jarvis, not a fork** — The router creates and configures a Jarvis instance internally. Users interact with the router, not Jarvis directly.

---

## API Surface

### Construction

```apl
router←⎕NEW Router
router.Handlers←myAppNamespace   ⍝ namespace where handler fns live
```

### Route Registration

Each HTTP method is a public instance method on the router. Signature:

```apl
router.{Method} path handlerName
router.{Method} path handlerName options
```

Where:
- `{Method}` — `Get`, `Post`, `Put`, `Delete`, `Patch`
- `path` — string, e.g. `'/users/{id}'` (curly-brace path params, matching OpenAPI convention)
- `handlerName` — string name of a function in `router.Handlers`
- `options` — (optional) APLAN namespace with route metadata

#### Minimal registration

```apl
router.Get  '/users'      'ListUsers'
router.Get  '/users/{id}' 'GetUser'
router.Post '/users'      'CreateUser'
```

#### With metadata (future-ready, stored now, consumed later)

```apl
router.Post '/users' 'CreateUser' (
    summary: 'Create a new user'
    description: 'Creates a user and returns the created resource.'
    tags: ('users'⋄)
    body: CreateUserBody
    response: (201 CreateUserResponse)
    errors: (
        (422 'Validation error')
        (409 'Email already exists')
    )
)
```

The router stores whatever is in `options` on the route descriptor — it does not interpret or validate it in the core phase. This is what makes the system extensible: future modules can read `route.options.body`, `route.options.tags`, etc.

### Starting the Server

```apl
router.Start 8080
```

Internally creates a Jarvis instance, sets `Paradigm←'REST'`, builds a `CodeLocation` namespace with `Get`/`Post`/`Put`/`Delete`/`Patch` methods that delegate to the router's dispatcher.

### Stopping the Server

```apl
router.Stop
```

---

## Internal Data Structures

### Route Descriptor

Each registered route becomes a namespace stored in `_routes`:

```
route.method    ←  'GET'                         ⍝ uppercase string
route.pattern   ←  '/users/{id}'                 ⍝ path with {param} placeholders
route.handler   ←  'GetUser'                     ⍝ fn name, resolved against Handlers
route.options   ←  ⎕NS ''                        ⍝ APLAN metadata (or empty ns if none given)
```

`_routes` is a vector of these namespaces, appended to on each registration call.

### Request Namespace (passed to handlers)

The router enriches the Jarvis request object before passing it to the handler:

```
req.Method       ←  'GET'                        ⍝ from Jarvis
req.Endpoint     ←  '/users/42'                  ⍝ from Jarvis
req.Payload      ←  ...                          ⍝ from Jarvis (parsed JSON body)
req.PathParams   ←  (id: '42')                   ⍝ set by router after path matching
req.QueryParams  ←  (q: 'search term')           ⍝ set by router from Jarvis query string
req.Headers      ←  ...                          ⍝ from Jarvis
```

---

## Core Components

### 1. Method Registration (`Get`, `Post`, `Put`, `Delete`, `Patch`)

Each is a thin wrapper that calls a shared `_Register` method:

```apl
∇ Get path handlerName
    _Register 'GET' path handlerName ⎕NS ''
∇

∇ Get path handlerName options
    _Register 'GET' path handlerName options
∇
```

Implementation note: Since Dyalog doesn't have method overloading in the traditional sense, the method will need to handle both the 2-argument case (path + handler) and the 3-argument case (path + handler + options). This can be done via a single method with optional right argument handling, or via the pattern used in the existing prototype where the registration method inspects its argument shape.

`_Register` creates the route descriptor namespace and appends it to `_routes`.

### 2. Path Matching (`_MatchPath`)

Carried forward from the existing prototype. Compares a pattern like `/users/{id}` against an incoming path like `/users/42`:

- Split both on `/`
- Segment count must match
- `{name}` segments capture the corresponding URL segment into `PathParams`
- Literal segments must match exactly (case-insensitive)

No changes needed from the existing implementation for the core phase.

**Future extension point:** Typed path params (`{id:int}`) could be added here later — the matcher would strip the type suffix, coerce the value, and feed the type into OpenAPI param schemas.

### 3. Dispatch (`_Dispatch`)

On each incoming request:

1. Filter `_routes` to those matching the request method
2. Iterate filtered routes, calling `_MatchPath` for each
3. On first match: enrich `req` with `PathParams` and `QueryParams`, call the handler
4. No match: return 404

This is the same algorithm as the existing prototype.

**Future extension points:**
- **Validation hook** — before calling the handler, check `route.options.body` and validate `req.Payload`
- **Middleware hook** — before/after the handler, run functions listed in `route.options.middleware`
- **Response validation** — after the handler, check the result against `route.options.response` (dev mode)
- **Error handling** — `:Trap` around the handler call, map signals to HTTP status codes

### 4. Server Lifecycle (`Start` / `Stop`)

`Start` creates the Jarvis bridge:

```apl
∇ Start port
    code←⎕NS ''
    code.inst←⎕THIS
    code.Get←{inst._Dispatch ⍵}
    code.Post←{inst._Dispatch ⍵}
    ⍝ ... etc for Put, Delete, Patch
    j←⎕NEW #.Jarvis
    j.Port←port
    j.Paradigm←'REST'
    j.CodeLocation←code
    j.Start
    _jarvis←j
∇
```

`Stop` calls `_jarvis.Stop`.

---

## Handler Contract

Handlers are regular APL functions in the `Handlers` namespace:

```apl
∇ result←GetUser req
    result←⎕NS ''
    result.id←req.PathParams.id
    result.name←'Jane Doe'
∇
```

- **Input:** `req` namespace (see Request Namespace above)
- **Output:** a namespace or scalar that Jarvis serializes to JSON
- Handlers do not need `:Access Public` unless they live inside a class (which they might — the `Handlers` namespace can be a class instance like in ExampleApp)

---

## File Structure

```
APLSource/
    Router.aplc          ⍝ The routing class (name TBD)
```

Single file. The router is self-contained. No external dependencies beyond Jarvis itself.

---

## Future Extension Points (not implemented now, but the design accommodates them)

| Feature | How it hooks in |
|---|---|
| **Schema validation** | Reads `route.options.body`, validates `req.Payload` before calling handler. Returns 422 on failure. |
| **OpenAPI generation** | Walks `_routes`, reads each `route.options` (summary, tags, body, response, errors, params), produces OpenAPI 3.0 JSON. |
| **Middleware** | Reads `route.options.middleware`, runs named functions before/after handler. Global middleware via `router.Use`. |
| **Route grouping** | `router.Group '/api/v1'` returns a sub-router that prepends the prefix to all registered paths. |
| **Error handling** | `:Trap` in `_Dispatch`, maps `⎕SIGNAL` EN values to HTTP status codes. Global error handler via `router.OnError`. |
| **Response validation** | Reads `route.options.response`, validates handler output in dev/debug mode. |
| **Typed path params** | `{id:int}` in path pattern — matcher coerces value and feeds type into OpenAPI. |
| **Schema helpers** | Utility functions (`String`, `Integer`, `Schema`) that produce APLAN/JSON Schema structures for use in `options.body`. |

All of these consume data that is already stored on `route.options` from the registration call. No registration API changes needed.

---

## What Changes From the Existing Prototype

| Aspect | Existing `FastJarvis.aplc` | New Router |
|---|---|---|
| Registration | `Add ('METHOD /path' 'handler')` — 2-element vector | `router.Post '/path' 'handler' options` — named method per HTTP verb, optional APLAN options |
| Route storage | `method`, `pattern`, `handler` fields | Same + `options` field for metadata |
| Comment scanning | ExampleApp uses `⍝@` + `⎕SRC` parsing | Dropped. Explicit registration only. |
| Constructor | Takes route pairs | No-arg constructor. Routes added via method calls. |
| Path matching | `_MatchPath` — works as-is | Carried forward unchanged |
| Dispatch | `_Dispatch` — works as-is | Carried forward, minor adjustments for new route structure |
