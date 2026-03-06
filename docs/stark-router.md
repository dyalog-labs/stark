# Stark router

Stark is the core class of the framework. It manages route registration, request dispatch, and OpenAPI spec generation.

## Fields

| Field        | Access  | Description                                                  |
|--------------|---------|--------------------------------------------------------------|
| `Handlers`   | Public  | Namespace (or class instance) where handler functions live   |
| `ThreadMode` | Public  | Controls Jarvis threading -- `''`, `0`, `1`, `'DEBUG'`, `'AUTO'` |
| `Info`       | Public  | Namespace with `title`, `version`, and optional `description` for the OpenAPI info block |

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
