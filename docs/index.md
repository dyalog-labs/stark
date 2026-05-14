# Stark

> **Not yet ready for production use.** Feedback is welcome — please open a GitHub issue if you have thoughts or run into problems.

Stark is a modern REST API framework for Dyalog APL. It provides a clean, Fastify-style routing layer on top of [Jarvis](jarvis-server.md), Dyalog's HTTP server library.

## Features

- **HTTP verb routing** -- Register handlers with `Get`, `Post`, `Put`, `Delete`, and `Patch`
- **Path parameters** -- Use `{param}` syntax for dynamic URL segments like `/users/{id}`
- **Query parameters** -- Access query string values through `req.QueryParams`
- **Automatic OpenAPI generation** -- Get a full OpenAPI 3.0.3 spec at `/openapi.json` with no extra work
- **Route metadata** -- Any valid OpenAPI operation field passes through to the spec; attach summaries, tags, request bodies, responses, security, and more
- **Root-level spec fields** -- Use `router.Spec` to add `components`, `security`, `servers`, and other root-level OpenAPI fields
- **Minimal boilerplate** -- Define a handler, register a route, start the server

## Quick look

```apl
router←⎕NEW Stark
router.Handlers←⎕THIS
router.Info←(title: 'My API' ⋄ version: '1.0.0')

'/items'      router.Get  'ListItems'
'/items/{id}' router.Get  'GetItem'
'/items'      router.Post ('CreateItem' createOpts)

router.Start 8080
```

## Requirements

- Dyalog APL 20.0 or later
- Conga networking library (ships with Dyalog)
- Jarvis (A copy is also provided in APLSource)

## Project structure

```
APLSource/
  Jarvis.aplc              ← HTTP server (Dyalog library)
  Stark.aplc               ← Router / framework
Examples/
  ExampleClass/
    ExampleApp.aplc        ← Sample CRUD API (class-based)
    Run.apls               ← Script to launch the example
  ExampleNonClass/
    APLSource/             ← Sample CRUD API (function-based)
    Run.apls               ← Script to launch the example
docs/                      ← This documentation
```
