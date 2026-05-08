# OpenAPI generation

Stark automatically generates an [OpenAPI 3.0.3](https://spec.openapis.org/oas/v3.0.3) specification from your registered routes and their metadata.

## Accessing the spec

When `router.Start` is called, Stark registers an internal route at `/openapi.json` that serves the spec as JSON.

```bash
curl http://localhost:8080/openapi.json
```

You can also get the spec programmatically:

```apl
spec←router.OpenAPI
```

## What gets included

The generated spec contains:

- **Info block** -- populated from `router.Info` (`title`, `version`, `description`)
- **Root-level fields** -- any extra OpenAPI root fields from `router.Spec` (`components`, `security`, `servers`, etc.)
- **Paths** -- one entry per registered route pattern
- **Operations** -- one per HTTP method on each path
- **Path parameters** -- automatically extracted from `{param}` segments in the route pattern

Internal handlers (function names starting with `_`) are excluded from the spec.

## Providing metadata

Pass a namespace as the second element of the route's right argument. Stark merges it into the OpenAPI operation object as-is — any valid [OpenAPI operation field](https://spec.openapis.org/oas/v3.0.3#operation-object) passes through untouched.

```apl
opts←(
    summary: 'Create a new item'
    tags: ('items'⋄)
    requestBody: (
        required: ⊂'true'
        content: (⍙application⍙47⍙json: (schema: (type: 'object' ⋄ required: 'name' 'price' ⋄ properties: (
            name:  (type: 'string')
            price: (type: 'number')
        ))))
    )
    responses: (
        201 (type: 'object' ⋄ properties: (
            id:    (type: 'integer')
            name:  (type: 'string')
            price: (type: 'number')
        ))
    )
)

'/items' router.Post ('CreateItem' opts)
```

Because field names must be valid APL names, use the `7162⌶` mangled form for keys containing special characters. `application/json` mangles to `⍙application⍙47⍙json`.

## Conveniences

Stark applies four automatic transformations on top of the pass-through:

| Convenience | Behaviour |
|-------------|-----------|
| `operationId` | Defaults to the handler function name if not provided |
| Path parameters | Auto-extracted from `{param}` URL segments; always `in: 'path'`, `required: true`, `schema: {type: 'string'}` |
| `responses` shorthand | A vector of `(statusCode schema)` pairs is converted to a mangled responses namespace |
| Default 200 | If no `responses` are specified, a generic `200 Successful response` entry is added |

**Everything else passes through untouched.** Use OpenAPI field names directly: `requestBody` (not `body`), `parameters`, `security`, `deprecated`, `externalDocs`, etc.

## Root-level OpenAPI fields

Use `router.Spec` to add fields at the root of the spec document alongside `openapi`, `info`, and `paths`:

```apl
router.Info←(title: 'My API' ⋄ version: '1.0.0')   ⍝ unchanged
router.Spec←(
    components: (securitySchemes: (bearerAuth: (type: 'http' ⋄ scheme: 'bearer')))
    security: ,⊂(bearerAuth: ⍬)
)
```

## Default behaviour

- Routes with no metadata still appear in the spec with a default `200 Successful response`.
- Path parameters are always generated with `in: 'path'`, `required: true`, and `schema: {type: 'string'}`. Supply a `parameters` entry in the route opts to override or augment.
- If no `responses` metadata is provided, a generic 200 response entry is added.

## Using with Swagger UI

Point Swagger UI (or any OpenAPI viewer) at your spec URL:

```
http://localhost:8080/openapi.json
```

This gives you interactive API documentation with no additional setup.
