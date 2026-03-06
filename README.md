# Stark

> **Internal project — not yet ready for public release.** I'm trying to gather feedback before opening this up more widely. I'd love to hear your thoughts....

Stark is a REST API framework for Dyalog APL. It provides Fastify-style routing on top of Jarvis.

## Features

- HTTP verb routing (`Get`, `Post`, `Put`, `Delete`, `Patch`)
- Path parameters (`/users/{id}`)
- Query parameter access
- Automatic OpenAPI 3.0.3 spec generation
- Route metadata (summaries, descriptions, tags, schemas)

## Quick Example

```apl
router←⎕NEW Stark
router.Handlers←⎕THIS

'/items'      router.Get  'ListItems'
'/items/{id}' router.Get  'GetItem'
'/items'      router.Post 'CreateItem'

router.Start 8080
```

## Requirements

- Dyalog APL 20.0+
- Conga (ships with Dyalog)

## Status

This project is in **active development**. APIs and behaviour may change. Feedback and bug reports are welcome.
