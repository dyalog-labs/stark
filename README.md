# Stark

> **Not yet ready for production use.** Feedback is welcome — please open a GitHub issue if you have thoughts or run into problems.


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
router.Register routes

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

Stark auto-generates an OpenAPI spec from your registered routes, served at `/openapi.json`. Route metadata — summaries, tags, request body schemas, response schemas — is passed alongside each route definition and merged directly into the OpenAPI operation object:

```apl
opts←(
    summary: 'Create an item'
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
routes←[
    'POST' '/items' 'CreateItem' opts
]
router.Register routes
```

Because APL names cannot contain `/`, content-type keys like `application/json` use the mangled form `⍙application⍙47⍙json`. The `responses` field accepts a shorthand vector of `(statusCode schema)` pairs, which Stark expands into full OpenAPI response objects. See the [OpenAPI docs](https://dyalog-labs.github.io/stark/openapi/) for the full reference.

This opens the door to Swagger UI, auto-generated client libraries (including APL clients), Postman collections, LLM tool integration, and any other tooling that consumes OpenAPI specs.

## Requirements

- Dyalog APL 20.0+
- Jarvis 1.22+ (installed automatically as a Tatin dependency)

## Installation

Stark is published on [Tatin](https://tatin.dev) as `dyalog_labs-Stark`:

```apl
]Tatin.LoadPackages [tatin]dyalog_labs-Stark #
```

## Documentation

Full documentation is at [dyalog-labs.github.io/stark](https://dyalog-labs.github.io/stark/). The source is in the [docs](docs/) folder.

## Status

This project is in **active development**. APIs and behaviour may change. Feedback and bug reports are welcome.
