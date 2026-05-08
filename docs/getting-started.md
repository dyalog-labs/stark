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

### 1. Get Stark and Jarvis

Download or clone both Stark and Jarvis into your project.

### 2. Import into the workspace

Load them however suits your project, for example:

```apl
]link.import # ./Path/To/Stark.aplc
]link.import # ./Path/To/Jarvis.aplc
```

### 3. Create a handler namespace

Handler functions receive a request object and return a result namespace.

```apl
∇ result←Hello req
  result←(message: 'Hello, world!')
∇
```

### 4. Create and configure the router

```apl
router←⎕NEW Stark
router.Handlers←⎕THIS
router.Info←(title: 'My API' ⋄ version: '0.1.0')
```

### 5. Register routes

```apl
'/'     router.Get 'Hello'
```

### 6. Start the server

```apl
router.Start 8080
```

Your API is now live at `http://localhost:8080/`. An OpenAPI spec is automatically available at `/openapi.json`.

### 7. Stop the server

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
      router←⎕NEW ##.Stark
      router.Handlers←⎕THIS
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

## ThreadMode

Control how Jarvis handles requests by setting `ThreadMode` before calling `Start`:

| Value     | Behaviour                                |
|-----------|------------------------------------------|
| `''`      | Jarvis default                           |
| `0`       | Run in thread 0 (main thread)            |
| `1`       | Run each request in a new thread         |
| `'DEBUG'` | Debug mode                               |
| `'AUTO'`  | Automatic thread management              |

```apl
router.ThreadMode←0
router.Start 8080
```
