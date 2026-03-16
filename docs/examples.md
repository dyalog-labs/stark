# Examples

## ExampleApp -- CRUD items API

The included `ExampleApp` class demonstrates a complete REST API with in-memory data storage.

### Source

Located at `Examples/ExampleApp.aplc` with a runner script at `Examples/Run.apls`.

### Running it

```bash
cd Examples
dyalogscript Run.apls
```

The server starts on port 8080.

### Endpoints

| Method   | Path                                     | Handler        | Description              |
|----------|------------------------------------------|----------------|--------------------------|
| `GET`    | `/`                                      | `Root`         | Health check             |
| `GET`    | `/items`                                 | `ListItems`    | List all items           |
| `GET`    | `/items/{id}`                            | `GetItem`      | Get item by ID           |
| `POST`   | `/items`                                 | `CreateItem`   | Create a new item        |
| `PUT`    | `/items/{id}`                            | `UpdateItem`   | Update an existing item  |
| `DELETE` | `/items/{id}`                            | `DeleteItem`   | Delete an item           |
| `GET`    | `/search?q=...`                          | `SearchItems`  | Search with query param  |
| `GET`    | `/customer/{cust_id}/invoice/{inv_id}`   | `GetInvoice`   | Multi-parameter path     |
| `GET`    | `/openapi.json`                          | *(auto)*       | OpenAPI spec             |

### Sample requests

```bash
# Health check
curl http://localhost:8080/

# List items
curl http://localhost:8080/items

# Get a specific item
curl http://localhost:8080/items/1

# Create an item
curl -X POST http://localhost:8080/items \
  -H 'Content-Type: application/json' \
  -d '{"name": "RIDE", "price": 0}'

# Update an item
curl -X PUT http://localhost:8080/items/1 \
  -H 'Content-Type: application/json' \
  -d '{"name": "Dyalog APL 20.0", "price": 0}'

# Delete an item
curl -X DELETE http://localhost:8080/items/2

# Search
curl 'http://localhost:8080/search?q=dyalog'

# Multi-parameter path
curl http://localhost:8080/customer/42/invoice/7

# OpenAPI spec
curl http://localhost:8080/openapi.json
```

### How it works

The constructor sets up the router and registers all routes:

```apl
router←⎕NEW ##.Stark
router.Handlers←⎕THIS         ⍝ handlers are methods on this class
router.Info←(title: 'Example Items API' ⋄ version: '1.0.0')

routes←[
    ⍝ Method    Endpoint                                Handler        Schema
      'GET'     '/'                                     'Root'         RootSchema
      'GET'     '/items'                                'ListItems'    ListItemsSchema
      'GET'     '/items/{id}'                           'GetItem'      GetItemSchema
      'POST'    '/items'                                'CreateItem'   CreateItemSchema
      'PUT'     '/items/{id}'                           'UpdateItem'   UpdateItemSchema
      'DELETE'  '/items/{id}'                           'DeleteItem'   DeleteItemSchema
      'GET'     '/search'                               'SearchItems'  SearchItemsSchema
      'GET'     '/customer/{cust_id}/invoice/{inv_id}'  'GetInvoice'   GetInvoiceSchema
]
router.Register routes
```

Each handler is a public method that receives the request and returns a namespace:

```apl
∇ result←GetItem req;id;matches
  :Access Public
  id←⊃2⊃⎕VFI req.PathParams.id    ⍝ convert string to number
  matches←(db.items.id=id)/db.items
  :If 0=≢matches
      req.Fail 404
      result←(detail: 'Item not found')
  :Else
      result←⊃matches
  :EndIf
∇
```

### Sample data

The app ships with two items pre-loaded:

| id | name       | price | tags                   |
|----|------------|-------|------------------------|
| 1  | Dyalog APL | 0     | `'language' 'array'`   |
| 2  | SharpPlot  | 0     | `'graphics' 'charting'`|
