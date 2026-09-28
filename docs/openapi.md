# OpenAPI generation

Stark automatically generates an [OpenAPI 3.0.3](https://spec.openapis.org/oas/v3.0.3) specification from your registered routes and their metadata.

## Accessing the spec

Every router has an internal route at `/openapi.json`, registered when the router is created, that serves the spec as JSON.

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

Because field names must be valid APL names, use the [`7162⌶` mangled form](stark-router.md#mangled-names) for keys containing special characters. `application/json` mangles to `⍙application⍙47⍙json`.

## Conveniences

Stark applies four automatic transformations on top of the pass-through:

| Convenience | Behaviour |
|-------------|-----------|
| `operationId` | Defaults to the handler function name if not provided |
| Path parameters | Generated from `{param}` URL segments with `in: 'path'`, `required: true`, `schema: {type: 'string'}`. A `parameters` entry you supply with the same `name` and `in: 'path'` replaces the generated one; other entries are added after the generated ones |
| `responses` shorthand | A vector of `(statusCode schema)` pairs is expanded into full response objects (see [below](#responses-shorthand)) |
| Default 200 | If no `responses` are specified, a generic `200 Successful response` entry is added |

**Everything else passes through untouched.** Use OpenAPI field names directly: `requestBody` (not `body`), `parameters`, `security`, `deprecated`, `externalDocs`, etc.

### Responses shorthand

Each `(statusCode schema)` pair becomes a response whose `description` is the standard HTTP status text and whose schema is served as `application/json`. Use `⍬` in place of a schema for a response with no body:

```apl
responses: (
    201 (type: 'object' ⋄ properties: (id: (type: 'integer')))
    204 ⍬
)
```

generates:

```json
"responses": {
  "201": {
    "description": "Created",
    "content": {"application/json": {"schema": {"type": "object", "properties": {"id": {"type": "integer"}}}}}
  },
  "204": {"description": "No Content"}
}
```

Status codes that Jarvis does not recognise get the description `Response`.

For full control — a custom description, headers, or other content types — pass `responses` as a namespace instead. It is used as-is, so write status codes in mangled form:

```apl
responses: (
    ⍙201: (description: 'Item created' ⋄ content: (⍙application⍙47⍙json: (schema: ItemSchema)))
)
```

## Overriding path parameters

Generated path parameters are always typed as strings. To document a different type, supply the parameter yourself:

```apl
opts←(parameters: ,⊂(name: 'id' ⋄ in: 'path' ⋄ required: ⊂'true' ⋄ schema: (type: 'integer')))
'/items/{id}' router.Get ('GetItem' opts)
```

This changes only the spec; `req.PathParams.id` is still a string.

## Root-level OpenAPI fields

Use `router.Spec` to add fields at the root of the spec document alongside `openapi`, `info`, and `paths`:

```apl
router.Info←(title: 'My API' ⋄ version: '1.0.0')   ⍝ unchanged
router.Spec←(
    components: (securitySchemes: (bearerAuth: (type: 'http' ⋄ scheme: 'bearer')))
    security: ,(bearerAuth: ⍬)
)
```

## Using with Swagger UI

Point Swagger UI (or any OpenAPI viewer) at your spec URL:

```
http://localhost:8080/openapi.json
```

This gives you interactive API documentation with no additional setup.
