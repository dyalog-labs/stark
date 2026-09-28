# Getting started

## Running the example app

The quickest way to try Stark is the included example application.

```bash
cd Examples/ExampleClass
dyalogscript Run.apls
```

This starts an HTTP server on port 8080 with several demo endpoints. Try it out:

```bash
curl http://localhost:8080/items
curl http://localhost:8080/items/1
curl -X POST http://localhost:8080/items -H 'Content-Type: application/json' -d '{"name":"RIDE","price":0}'
curl http://localhost:8080/openapi.json
```

## Creating your own application

### 1. Install Stark

Stark is published on the [Tatin](https://tatin.dev) registry as `dyalog_labs-Stark`. Jarvis is a dependency and is installed with it.

To load Stark into the workspace:

```apl
]Tatin.LoadPackages [tatin]dyalog_labs-Stark #
```

To install it into a project and load it from there:

```apl
]Tatin.InstallPackages [tatin]dyalog_labs-Stark ./packages
]Tatin.LoadDependencies ./packages #
```

Either way, `#.Stark` refers to the Stark class.

??? note "Without Tatin"
    Copy `Stark.aplc` and `Jarvis.aplc` from this repository's `APLSource` folder and import them **into the same namespace**. Stark looks for Jarvis alongside itself, so the two must be siblings:

    ```apl
    ]link.import # ./APLSource/Stark.aplc
    ]link.import # ./APLSource/Jarvis.aplc
    ```

### 2. Write a handler

A handler takes the request as its right argument and returns the response body, usually a namespace, which Jarvis serializes to JSON.

```apl
∇ result←Hello req
  result←(message: 'Hello, world!')
∇
```

### 3. Create and configure the router

```apl
router←Stark.New ()
router.Info←(title: 'My API' ⋄ version: '0.1.0')
```

Stark looks for handler functions in `router.Handlers`, which defaults to the namespace that called `Stark.New`. If your handlers live elsewhere, set it explicitly:

```apl
router.Handlers←#.MyHandlers
```

### 4. Register routes

```apl
'/' router.Get 'Hello'
```

### 5. Start the server

```apl
router.Start 8080
```

Your API is now live at `http://localhost:8080/`. An OpenAPI spec is automatically available at `/openapi.json`.

### 6. Stop the server

```apl
router.Stop
```

## Using a class

For larger applications, define a class that owns the router. This keeps handlers, data, and configuration together. See the [Examples](examples.md) page for a full class-based application.

```apl
:Class MyApp
    :Field Private router

    ∇ Make
      :Access Public
      :Implements Constructor
      router←##.Stark.New ()    ⍝ Handlers defaults to this instance
      '/ping' router.Get 'Ping'
    ∇

    ∇ Start port
      :Access Public
      router.Start port
    ∇

    ∇ Stop
      :Access Public
      router.Stop
    ∇

    ∇ result←Ping req
      :Access Public
      result←(status: 'pong')
    ∇
:EndClass
```

!!! note
    Handler methods must be `:Access Public`. Stark calls them from outside the class, so private methods are not visible to it.

## Next steps

- Choose how the server runs relative to your session with [`ThreadMode`](stark-router.md#thread-mode).
- Register many routes at once with [`Register`](stark-router.md#bulk-registration).
- Describe your routes for the [OpenAPI spec](openapi.md).
