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
- **Paths** -- one entry per registered route pattern
- **Operations** -- one per HTTP method on each path
- **Path parameters** -- automatically extracted from `{param}` segments in the route pattern
- **Request body schemas** -- from the `body` metadata key
- **Response schemas** -- from the `response` metadata key
- **Error responses** -- from the `errors` metadata key
- **Summaries, descriptions, and tags** -- from route metadata

Internal handlers (function names starting with `_`) are excluded from the spec.

## Providing metadata

Attach metadata when registering a route by passing a two-element vector:

```apl
opts←(
    summary: 'Create a new item'
    tags: ('items'⋄)
    body: (type: 'object' ⋄ required: 'name' 'price' ⋄ properties: (
        name:  (type: 'string')
        price: (type: 'number')
    ))
    response: (201 (type: 'object' ⋄ properties: (
        id:    (type: 'integer')
        name:  (type: 'string')
        price: (type: 'number')
    )))
    errors: ((404 'Not found')⋄)
)

'/items' router.Post ('CreateItem' opts)
```

## Metadata reference

| Key           | Type                          | Maps to OpenAPI                          |
|---------------|-------------------------------|------------------------------------------|
| `summary`     | String                        | `operation.summary`                      |
| `description` | String                        | `operation.description`                  |
| `tags`        | Vector of strings             | `operation.tags`                         |
| `body`        | Schema namespace              | `operation.requestBody.content.application/json.schema` |
| `response`    | `(statusCode schema)`         | `operation.responses.{code}.content.application/json.schema` |
| `errors`      | Vector of `(code description)` | `operation.responses.{code}.description` |

## Default behaviour

- Routes with no metadata still appear in the spec with a default `200 Successful response`.
- Path parameters are always generated with `in: 'path'`, `required: true`, and `schema: {type: 'string'}`.
- If no response metadata is provided, a generic 200 response entry is added.

## Using with Swagger UI

Point Swagger UI (or any OpenAPI viewer) at your spec URL:

```
http://localhost:8080/openapi.json
```

This gives you interactive API documentation with no additional setup.
