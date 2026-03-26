# Stark

> **Internal project — not yet ready for public release.** I'm trying to gather feedback before opening this up more widely. I'd love to hear your thoughts. Although this project is "publicly" available on test.tatin.dev, it is not really meant for public use yet. 


Stark is a routing layer on top of Jarvis that makes it straightforward to build REST APIs in Dyalog APL. Each route maps directly to a function, and removes the need for manual path parsing. Simply define your routes, write your handlers, and Stark takes care of dispatch. It also auto-generates an OpenAPI spec from your route definitions, giving you machine-readable documentation, Swagger UI, client generation, and tool integration for free.

## Why REST?

Jarvis JSON mode is simple and works well when both sides are APL. But if your API is consumed by external clients, such as web frontends, mobile apps, or other languages, REST is the interface they expect. Standard HTTP verbs, predictable URLs, and proper status codes make your service immediately usable by any HTTP client without custom documentation.

## What Stark adds to Jarvis

With raw Jarvis in REST mode, you write one function per HTTP verb that handles every request for that verb. Routing is up to you, and you must manually split the path, match the segments, and extract path parameters:

```apl
∇ result←Get request;segments;root;id
  segments←'/'(≠⊆⊢)request.Endpoint
  root←⊃segments
  :Select root
  :Case 'items'
      :If 1=≢segments
          result←ListItems request
      :Else
          id←2⊃segments
          result←GetItem id request
      :EndIf
  :Case 'health'
      result←Health request
  :EndSelect
∇
```

This works, but as endpoints grow, the routing logic can get deeply nested and harder to maintain. Stark replaces this with declarative route registration:

```apl
router←Stark.New ()
routes←[
    'GET' '/health'     'Health'
    'GET' '/items'      'ListItems'
    'GET' '/items/{id}' 'GetItem'
]
routes←router.Register routes

router.Start 8080
```

Same three endpoints, but each route is a single line and each handler is a plain function. Stark handles path matching, parameter extraction, and dispatch. Your handler receives a request object with `PathParams` and `QueryParams` already parsed:

```apl
∇ result←GetItem request
  id←request.PathParams.id
  ⍝ look up and return the item
∇
```

## OpenAPI spec generation

Stark auto-generates an OpenAPI spec from your registered routes, served at `/openapi.json`. You can add metadata, such as summaries, descriptions, tags, and request/response schemas, alongside each route definition:

```apl
opts←(summary: 'Get an item by ID' ⋄ tags: ('items'⋄))
routes←[
    'GET' '/items/{id}' 'GetItem' opts
]
routes←router.Register routes
```

This opens the door to Swagger UI, auto-generated client libraries (including APL clients), Postman collections, LLM tool integration, and any other tooling that consumes OpenAPI specs.

## Requirements

- Dyalog APL 20.0+
- Jarvis 1.22+ (Included in the Tatin package)

## Documentation

See the [docs](docs/) folder for full reference documentation.

## Status

This project is in **active development**. APIs and behaviour may change. Feedback and bug reports are welcome.
