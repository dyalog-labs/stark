# REST API Router Patterns Across Languages & Frameworks

A comprehensive review of how REST API routers declare routes, attach metadata/schema, handle validation, and generate OpenAPI specs — to inform the design of a router for Dyalog Jarvis.

---

## Table of Contents

1. [Executive Summary & Key Takeaways](#executive-summary)
2. [Python — FastAPI](#fastapi)
3. [Python — Flask + Flask-Smorest](#flask)
4. [Node.js — Express](#express)
5. [Node.js — Fastify](#fastify)
6. [Node.js — Hono](#hono)
7. [Java — Spring Boot](#spring-boot)
8. [C# — ASP.NET Core Minimal APIs](#aspnet)
9. [Go — Gin](#gin)
10. [Go — Echo](#echo)
11. [Rust — Actix-web](#actix)
12. [Rust — Axum](#axum)
13. [Ruby — Rails](#rails)
14. [Ruby — Grape](#grape)
15. [Elixir — Phoenix](#phoenix)
16. [PHP — Laravel](#laravel)
17. [Comparative Analysis](#comparative-analysis)
18. [Ideas for Dyalog Jarvis](#recommendations)

---

## 1. Executive Summary & Key Takeaways <a name="executive-summary"></a>

After surveying 15+ frameworks across 8 languages, a few dominant patterns emerge:

**Route declaration** falls into three families:
- **Decorator/Annotation-based** (FastAPI, Flask, Spring, Actix): You annotate a function with route metadata. The function *is* the handler.
- **Builder/Fluent-based** (Express, Fastify, Gin, Echo, Hono): You call methods on a router object to register routes imperatively.
- **DSL/Config-based** (Rails, Phoenix, Laravel): A dedicated routing DSL maps paths to controllers.

**Schema & validation** falls into two camps:
- **Schema-first** (Fastify, FastAPI, Spring): Schema is declared *with* the route, validation happens automatically before the handler runs, and errors are returned in a standardized format. This is the modern best practice.
- **Schema-separate** (Express, Gin, Rails): Schema/validation is bolted on via middleware or separate libraries. This is more flexible but less cohesive.

**OpenAPI generation** is either:
- **Automatic from route+schema metadata** (FastAPI, Fastify, Spring, ASP.NET, Hono): The framework has enough information from route declarations to produce a spec.
- **Requires a plugin or manual work** (Express, Gin, Rails, Phoenix): You either annotate additionally or maintain the spec by hand.

**The gold standard** for what you're trying to build is **FastAPI**. It nails the trifecta: route declaration is minimal, schema is derived from types, validation is automatic, and OpenAPI is generated with zero extra work. Fastify is the runner-up, taking a JSON Schema approach instead of type-derived schemas.

---

## 2. Python — FastAPI <a name="fastapi"></a>

FastAPI is widely considered the best example of "routes + schema + validation + OpenAPI" done right. It's the strongest model for what you want to build.

### Route Declaration

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="My API", version="1.0.0")

# Simple GET with path parameter
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

# POST with request body schema
class CreateUser(BaseModel):
    name: str = Field(..., min_length=1, max_length=100, description="The user's full name")
    email: str = Field(..., description="Valid email address")
    age: int | None = Field(None, ge=0, le=150, description="Age in years")

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.post("/users", response_model=UserResponse, status_code=201,
          summary="Create a new user",
          description="Creates a user account and returns the created resource.",
          tags=["users"])
async def create_user(user: CreateUser):
    # `user` is already validated — if the body was invalid,
    # FastAPI returned a 422 with detailed errors before this runs.
    return {"id": 1, "name": user.name, "email": user.email}
```

### What metadata can you attach to a route?

| Metadata | How it's expressed |
|---|---|
| HTTP method | Decorator name (`@app.get`, `@app.post`, etc.) |
| Path & path params | String arg + function param type hints |
| Query params | Function params not in path, with type hints |
| Request body schema | Pydantic model as a function parameter |
| Response schema | `response_model=` kwarg |
| Status code | `status_code=` kwarg |
| Summary / Description | `summary=`, `description=` kwargs |
| Tags (grouping) | `tags=["users"]` |
| Deprecation | `deprecated=True` |
| Dependencies (auth, etc.) | `dependencies=[Depends(verify_token)]` |

### Validation

Validation is **automatic and happens before the handler**. FastAPI uses Pydantic under the hood. If a request doesn't match the schema, the client gets a 422 response with a structured error body:

```json
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

The handler never executes if validation fails. This is a critical design choice: **the router owns validation**.

### OpenAPI Generation

Automatic. Visit `/docs` (Swagger UI) or `/openapi.json` and you get a full OpenAPI 3.0 spec derived from your route decorators + Pydantic models. Zero additional configuration.

### Key Design Insight

FastAPI's magic comes from **Python type hints as the single source of truth**. The path `"/users/{user_id}"` combined with `user_id: int` tells FastAPI everything: the parameter name, where it comes from, what type it is, and how to validate it. The Pydantic model for the body adds field-level constraints and descriptions. The `response_model` tells it the output shape. All of this feeds directly into OpenAPI generation.

---

## 3. Python — Flask + Flask-Smorest <a name="flask"></a>

Flask itself is minimalist — routing is bare-bones. Flask-Smorest (or the older Flask-RESTX) adds schema and OpenAPI support.

### Route Declaration (vanilla Flask)

```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    return {"user_id": user_id}
```

Flask's routing is simple: a path string with typed converters (`<int:user_id>`, `<string:name>`) and a `methods` list. No schema, no validation, no OpenAPI.

### Route Declaration (Flask-Smorest — adds schema + OpenAPI)

```python
from flask_smorest import Blueprint, abort
from marshmallow import Schema, fields

blp = Blueprint("users", __name__, url_prefix="/users",
                description="Operations on users")

class CreateUserSchema(Schema):
    name = fields.String(required=True, metadata={"description": "Full name"})
    email = fields.String(required=True)

class UserSchema(Schema):
    id = fields.Integer()
    name = fields.String()
    email = fields.String()

@blp.route("/")
class Users(MethodView):
    @blp.arguments(CreateUserSchema)      # validates request body
    @blp.response(201, UserSchema)        # documents response
    def post(self, user_data):
        # user_data is already validated and deserialized
        return {"id": 1, **user_data}
```

### Validation

With Flask-Smorest, validation is done by Marshmallow schemas and happens before the handler. Without it, Flask does no validation at all — you'd manually parse `request.json` and check fields yourself.

### OpenAPI

Flask-Smorest generates OpenAPI 3.0 automatically from the `@blp.arguments` and `@blp.response` decorators. Vanilla Flask has no built-in OpenAPI support.

### Key Design Insight

Flask shows the **layered approach**: a minimal router that knows nothing about schemas, with schema+validation+OpenAPI bolted on through extensions. This works but creates a split where the route declaration and the schema declaration are separate concerns that must be kept in sync.

---

## 4. Node.js — Express <a name="express"></a>

Express is the most widely-used Node.js framework. Its router is extremely minimal.

### Route Declaration

```javascript
const express = require('express');
const app = express();
const router = express.Router();

// Basic route
router.get('/users/:userId', (req, res) => {
    const userId = req.params.userId;  // always a string!
    res.json({ userId });
});

// Route with middleware chain
router.post('/users',
    validateBody(createUserSchema),  // custom middleware
    authorize('admin'),               // custom middleware
    (req, res) => {
        res.status(201).json(req.body);
    }
);

// Mount the router
app.use('/api/v1', router);
```

### What metadata can you attach to a route?

Very little natively. Express routes have:
- A **path pattern** (with `:param` syntax for path params)
- An **HTTP method**
- A **chain of middleware/handlers**

That's it. No schema, no types, no descriptions. Everything else is done via middleware (like `express-validator`, `joi`, `zod`, `celebrate`, etc.).

### Validation (via Zod, typical modern approach)

```javascript
const { z } = require('zod');

const createUserSchema = z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().min(0).max(150).optional(),
});

// Middleware factory
function validateBody(schema) {
    return (req, res, next) => {
        const result = schema.safeParse(req.body);
        if (!result.success) {
            return res.status(422).json({ errors: result.error.issues });
        }
        req.body = result.data;
        next();
    };
}
```

### OpenAPI

Express has **no built-in OpenAPI support**. You either use `swagger-jsdoc` (JSDoc comments → spec), `express-openapi-validator` (spec → validation), or maintain the spec manually. This is the fundamental weakness of Express for your use case.

### Key Design Insight

Express demonstrates the **"minimal core, middleware for everything"** philosophy. The router itself is just a pattern matcher that dispatches to handler chains. All concerns (parsing, validation, auth, logging) are middleware. This is maximally flexible but means there's no single place that knows enough about a route to generate documentation.

---

## 5. Node.js — Fastify <a name="fastify"></a>

Fastify is the most relevant Node.js framework for your goals. It was designed with schema-first routing and OpenAPI generation as core features.

### Route Declaration

```javascript
const fastify = require('fastify')();

// Schema-first route declaration
fastify.post('/users', {
    schema: {
        summary: 'Create a new user',
        description: 'Creates a user and returns the created resource.',
        tags: ['users'],
        body: {
            type: 'object',
            required: ['name', 'email'],
            properties: {
                name:  { type: 'string', minLength: 1, maxLength: 100,
                         description: 'Full name' },
                email: { type: 'string', format: 'email' },
                age:   { type: 'integer', minimum: 0, maximum: 150 },
            }
        },
        response: {
            201: {
                type: 'object',
                properties: {
                    id:    { type: 'integer' },
                    name:  { type: 'string' },
                    email: { type: 'string' },
                }
            }
        },
        params: {
            // for routes with path params
        },
        querystring: {
            // for query parameters
        }
    },
    handler: async (request, reply) => {
        // request.body is already validated and coerced
        reply.code(201).send({ id: 1, ...request.body });
    }
});
```

### What metadata can you attach to a route?

| Metadata | How it's expressed |
|---|---|
| HTTP method | Method name on fastify object |
| Path & path params | String + `schema.params` |
| Query params | `schema.querystring` |
| Request body schema | `schema.body` (JSON Schema) |
| Response schema(s) | `schema.response` keyed by status code |
| Headers schema | `schema.headers` |
| Summary / Description | `schema.summary`, `schema.description` |
| Tags | `schema.tags` |
| Security | via `@fastify/swagger` decorators |

### Validation

Fastify compiles JSON Schemas into optimized validation functions using `ajv` at startup time. Validation happens automatically before the handler. Invalid requests get a structured error response. Notably, **response schemas also act as serialization filters** — properties not in the response schema are stripped from the output, which prevents accidental data leakage.

### OpenAPI

With the `@fastify/swagger` plugin, OpenAPI 3.0 specs are generated automatically from route schemas. The plugin reads all registered routes and their schemas at startup and produces the spec. No additional annotations needed.

### Key Design Insight

Fastify takes a **"JSON Schema is the universal language"** approach. Since JSON Schema is already the schema language of OpenAPI, there's a direct 1:1 mapping from route schemas to the generated spec. This avoids any translation layer. The tradeoff is that JSON Schema is more verbose than type-annotation approaches (like FastAPI's), but it's more explicit and language-agnostic.

---

## 6. Node.js — Hono <a name="hono"></a>

Hono is a modern, lightweight framework that runs everywhere (Node, Deno, Bun, Cloudflare Workers). Its `@hono/zod-openapi` package is notable.

### Route Declaration (with Zod OpenAPI)

```typescript
import { OpenAPIHono, createRoute, z } from '@hono/zod-openapi';

const createUserRoute = createRoute({
    method: 'post',
    path: '/users',
    tags: ['users'],
    summary: 'Create a new user',
    request: {
        body: {
            content: {
                'application/json': {
                    schema: z.object({
                        name: z.string().min(1).openapi({ description: 'Full name' }),
                        email: z.string().email(),
                    })
                }
            }
        }
    },
    responses: {
        201: {
            content: {
                'application/json': {
                    schema: z.object({
                        id: z.number(),
                        name: z.string(),
                    })
                }
            },
            description: 'User created',
        }
    }
});

const app = new OpenAPIHono();

app.openapi(createUserRoute, async (c) => {
    const body = c.req.valid('json');  // typed and validated
    return c.json({ id: 1, name: body.name }, 201);
});

// Generate OpenAPI spec
app.doc('/openapi.json', { openapi: '3.0.0', info: { title: 'My API', version: '1.0.0' } });
```

### Key Design Insight

Hono separates **route definition** from **handler registration**. You first create a route object that fully describes the contract (path, method, schemas, docs), then separately register a handler for that route. This means the route definition is a **standalone, inspectable data structure** — which is very powerful for tooling, testing, and spec generation. The route is data, not just a side effect of calling a function.

---

## 7. Java — Spring Boot <a name="spring-boot"></a>

Spring Boot uses annotation-based routing, which is the Java ecosystem's standard pattern.

### Route Declaration

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "Users", description = "User management operations")
public class UserController {

    @Operation(
        summary = "Create a new user",
        description = "Creates a user and returns the created resource"
    )
    @ApiResponse(responseCode = "201", description = "User created",
        content = @Content(schema = @Schema(implementation = UserResponse.class)))
    @ApiResponse(responseCode = "422", description = "Validation error")
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // request is already validated by @Valid
        return ResponseEntity.status(201).body(new UserResponse(...));
    }

    @GetMapping("/{userId}")
    public UserResponse getUser(
            @PathVariable Long userId,
            @RequestParam(required = false, defaultValue = "false") Boolean verbose) {
        return new UserResponse(...);
    }
}

// Schema as a class with validation annotations
public record CreateUserRequest(
    @NotBlank @Size(max = 100)
    @Schema(description = "The user's full name", example = "Jane Doe")
    String name,

    @NotBlank @Email
    @Schema(description = "Valid email address")
    String email,

    @Min(0) @Max(150)
    @Schema(description = "Age in years", nullable = true)
    Integer age
) {}
```

### Validation

Spring uses **Bean Validation (JSR 380)** annotations (`@NotBlank`, `@Size`, `@Min`, `@Email`, etc.) on the DTO classes. The `@Valid` annotation on the controller parameter triggers validation before the handler runs. Validation errors result in a `MethodArgumentNotValidException` that's typically mapped to a 422 response.

### OpenAPI

SpringDoc (or the older SpringFox) scans all `@RestController` classes, reads annotations (`@Operation`, `@Schema`, `@ApiResponse`, etc.), and generates an OpenAPI 3.0 spec. It's automatic but requires the OpenAPI annotations to get rich documentation.

### Key Design Insight

Spring demonstrates the **"annotations everywhere"** approach. The route, schema, validation, and documentation metadata are all expressed as annotations on classes and methods. This is very powerful but can feel heavy — a single endpoint might have 10+ annotations. The benefit is that everything about an endpoint is co-located.

---

## 8. C# — ASP.NET Core Minimal APIs <a name="aspnet"></a>

ASP.NET Core's Minimal APIs (introduced in .NET 6) moved away from controller-based routing to a more concise, fluent style.

### Route Declaration

```csharp
var app = WebApplication.Create(args);

// Simple route
app.MapGet("/users/{userId:int}", (int userId) => new { userId });

// Route with full metadata chain
app.MapPost("/users", ([FromBody] CreateUserRequest request) =>
        Results.Created($"/users/{1}", new UserResponse(1, request.Name, request.Email)))
    .WithName("CreateUser")
    .WithSummary("Create a new user")
    .WithDescription("Creates a user and returns the created resource.")
    .WithTags("users")
    .Accepts<CreateUserRequest>("application/json")
    .Produces<UserResponse>(StatusCodes.Status201Created)
    .Produces<ValidationProblem>(StatusCodes.Status422UnprocessableEntity)
    .RequireAuthorization("AdminPolicy");

// Schema as a record with validation attributes
public record CreateUserRequest(
    [Required, StringLength(100, MinimumLength = 1)] string Name,
    [Required, EmailAddress] string Email,
    [Range(0, 150)] int? Age
);
```

### Validation

ASP.NET supports `DataAnnotations` (`[Required]`, `[Range]`, `[EmailAddress]`, etc.) and also FluentValidation for more complex scenarios. For Minimal APIs, validation must be explicitly wired up (it's not as automatic as in controller-based ASP.NET, where `[ApiController]` triggers it). The ecosystem is moving toward making this more automatic.

### OpenAPI

ASP.NET 9+ has **built-in OpenAPI generation** via `Microsoft.AspNetCore.OpenApi`. The `.Produces<T>()`, `.Accepts<T>()`, `.WithSummary()` etc. fluent methods feed directly into the generated spec.

### Key Design Insight

ASP.NET Minimal APIs show a **fluent builder pattern** for attaching metadata to routes. Instead of annotations on a class, you chain methods after the route registration. This is particularly interesting because it means the metadata attachment is imperative and composable — you could write helper functions that add standard metadata to routes.

---

## 9. Go — Gin <a name="gin"></a>

Gin is Go's most popular web framework. Go's type system and lack of annotations mean routing works differently.

### Route Declaration

```go
func main() {
    r := gin.Default()

    users := r.Group("/users")
    {
        users.GET("/:userId", getUser)
        users.POST("/", createUser)
    }

    r.Run(":8080")
}

type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=1,max=100"`
    Email string `json:"email" binding:"required,email"`
    Age   *int   `json:"age" binding:"omitempty,min=0,max=150"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(422, gin.H{"error": err.Error()})
        return
    }
    c.JSON(201, gin.H{"id": 1, "name": req.Name, "email": req.Email})
}
```

### Validation

Gin uses struct tags (`binding:"required,email"`) powered by the `go-playground/validator` library. However, validation is **not automatic** — you must explicitly call `ShouldBindJSON` in the handler. If you forget, there's no validation. The router itself doesn't know about the schema.

### OpenAPI

Gin has **no built-in OpenAPI support**. Libraries like `swaggo/gin-swagger` parse Go doc comments and struct tags to generate specs, but it's a separate tooling concern:

```go
// @Summary Create a new user
// @Tags users
// @Accept json
// @Produce json
// @Param request body CreateUserRequest true "User to create"
// @Success 201 {object} UserResponse
// @Router /users [post]
func createUser(c *gin.Context) { ... }
```

These are Go comments, not code. They can drift from reality.

### Key Design Insight

Gin illustrates a challenge that's directly relevant to APL/Dyalog: **when your language doesn't have annotations or decorators, metadata must live somewhere else**. In Gin's case, it's split between struct tags (for validation) and doc comments (for OpenAPI). Neither is checked by the compiler. This is the fundamental tension you'll face with Jarvis.

---

## 10. Go — Echo <a name="echo"></a>

Echo is similar to Gin but worth noting for its slightly different validation approach.

### Route Declaration

```go
e := echo.New()

e.POST("/users", createUser)
e.GET("/users/:id", getUser)

// Route-level middleware
admin := e.Group("/admin", adminMiddleware)
admin.DELETE("/users/:id", deleteUser)
```

### Validation

Echo has a `Validator` interface you can plug in globally:

```go
type CustomValidator struct {
    validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.validator.Struct(i)
}

e.Validator = &CustomValidator{validator: validator.New()}

// Then in handlers:
func createUser(c echo.Context) error {
    u := new(CreateUserRequest)
    if err := c.Bind(u); err != nil {
        return err
    }
    if err := c.Validate(u); err != nil {  // explicit call
        return echo.NewHTTPError(422, err.Error())
    }
    // ...
}
```

### Key Design Insight

Echo shows the **pluggable validator** pattern — the framework defines an interface, you provide the implementation. This keeps the framework agnostic about validation libraries while still giving a standard hook point. For Jarvis, this could mean defining a validation interface that users can implement with whatever logic they want.

---

## 11. Rust — Actix-web <a name="actix"></a>

Rust frameworks leverage the type system heavily for routing.

### Route Declaration

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Deserialize, Validate)]
struct CreateUser {
    #[validate(length(min = 1, max = 100))]
    name: String,
    #[validate(email)]
    email: String,
    #[validate(range(min = 0, max = 150))]
    age: Option<i32>,
}

#[derive(Serialize)]
struct UserResponse {
    id: i64,
    name: String,
    email: String,
}

#[actix_web::post("/users")]
async fn create_user(body: web::Json<CreateUser>) -> HttpResponse {
    // Deserialization happens automatically via the `web::Json` extractor.
    // If the JSON doesn't match CreateUser, a 400 is returned automatically.
    let user = body.into_inner();
    user.validate().unwrap(); // validation is a separate step
    HttpResponse::Created().json(UserResponse {
        id: 1,
        name: user.name,
        email: user.email,
    })
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(create_user)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### Key Design Insight

Actix-web uses Rust's **extractor pattern**: the function signature itself declares what the handler needs. `web::Json<CreateUser>` means "parse the body as JSON into a `CreateUser` struct." `web::Path<(i64,)>` means "extract path parameters." The framework uses Rust's type system to determine extraction behavior at compile time. This is the compile-time equivalent of FastAPI's type-hint approach.

---

## 12. Rust — Axum <a name="axum"></a>

Axum takes the extractor pattern further with a more composable design.

### Route Declaration

```rust
use axum::{
    routing::{get, post},
    Router, Json, extract::Path,
};

let app = Router::new()
    .route("/users", post(create_user))
    .route("/users/:user_id", get(get_user))
    .layer(middleware_stack);

async fn create_user(Json(payload): Json<CreateUser>) -> (StatusCode, Json<UserResponse>) {
    (StatusCode::CREATED, Json(UserResponse { ... }))
}

async fn get_user(Path(user_id): Path<i64>) -> Json<UserResponse> {
    Json(UserResponse { ... })
}
```

### Key Design Insight

Axum's routing is **entirely based on function signatures**. There are no macros or annotations on the handler — the router is built separately via `Router::new().route(...)`, and the handler's type signature determines how the request is parsed. This creates a very clean separation between "what path maps to what handler" (the router) and "what the handler needs from the request" (the function signature).

---

## 13. Ruby — Rails <a name="rails"></a>

Rails uses a dedicated routing DSL, completely separate from controllers.

### Route Declaration

```ruby
# config/routes.rb — the routing DSL
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:index, :show, :create, :update, :destroy] do
        member do
          post :activate
        end
        collection do
          get :search
        end
      end
    end
  end
end
```

This single `resources :users` call generates 5 standard routes:

| Method | Path | Controller#Action |
|---|---|---|
| GET | /api/v1/users | users#index |
| GET | /api/v1/users/:id | users#show |
| POST | /api/v1/users | users#create |
| PATCH | /api/v1/users/:id | users#update |
| DELETE | /api/v1/users/:id | users#destroy |

### Validation

Rails validates in the **Model layer**, not the router or controller:

```ruby
class User < ApplicationRecord
  validates :name, presence: true, length: { maximum: 100 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :age, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true
end
```

### OpenAPI

Rails has no built-in OpenAPI support. Gems like `rswag` or `rspec-openapi` generate specs from tests or special RSpec DSLs.

### Key Design Insight

Rails shows the **convention-over-configuration** approach: `resources :users` generates RESTful routes by convention. The router is a high-level DSL that knows about REST resource patterns, not just individual path→handler mappings. However, the router knows *nothing* about schemas — that concern lives entirely in the model layer.

---

## 14. Ruby — Grape <a name="grape"></a>

Grape is a REST API framework for Ruby that's much more schema-aware than Rails.

### Route Declaration

```ruby
class UsersAPI < Grape::API
  resource :users do
    desc 'Create a user',
         summary: 'Create a new user',
         success: { code: 201, model: Entities::User },
         failure: [{ code: 422, message: 'Validation error' }],
         tags: ['users']
    params do
      requires :name, type: String, desc: 'Full name',
               values: { length: 1..100 }
      requires :email, type: String, desc: 'Email address',
               regexp: URI::MailTo::EMAIL_REGEXP
      optional :age, type: Integer, desc: 'Age in years',
               values: 0..150
    end
    post do
      # params are already validated
      User.create!(declared(params))
    end
  end
end
```

### OpenAPI

With `grape-swagger`, OpenAPI specs are generated from the `desc` and `params` blocks automatically.

### Key Design Insight

Grape is interesting because it uses a **block-based DSL for parameter schemas** that's co-located with the route. The `params do ... end` block is both the validation spec and the documentation source. This is a good middle-ground pattern for languages without type annotations.

---

## 15. Elixir — Phoenix <a name="phoenix"></a>

Phoenix uses a pipeline-based router with explicit scope/pipe composition.

### Route Declaration

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
    plug MyAppWeb.AuthPlug
  end

  scope "/api/v1", MyAppWeb do
    pipe_through :api

    resources "/users", UserController, only: [:index, :show, :create]
    get "/users/search", UserController, :search
  end
end
```

### Validation

Phoenix uses **Ecto changesets** for validation, which happens in the context layer:

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:name, :email, :age])
  |> validate_required([:name, :email])
  |> validate_length(:name, max: 100)
  |> validate_format(:email, ~r/@/)
  |> validate_number(:age, greater_than_or_equal_to: 0)
end
```

### OpenAPI

Libraries like `open_api_spex` let you define schemas and attach them to controller actions. It's not automatic from the router.

### Key Design Insight

Phoenix's **pipeline concept** is powerful: you define named pipelines of middleware (plugs), then apply them to groups of routes. This is a clean way to apply cross-cutting concerns (auth, content negotiation, rate limiting) to route groups without repeating them on every route.

---

## 16. PHP — Laravel <a name="laravel"></a>

Laravel has a expressive routing DSL with several approaches to validation.

### Route Declaration

```php
// routes/api.php
Route::prefix('v1')->group(function () {
    Route::apiResource('users', UserController::class);

    Route::get('/users/search', [UserController::class, 'search'])
        ->middleware('throttle:60,1')
        ->name('users.search');
});
```

### Validation (via Form Requests)

```php
class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'  => 'required|string|max:100',
            'email' => 'required|email',
            'age'   => 'nullable|integer|min:0|max:150',
        ];
    }

    public function messages(): array
    {
        return [
            'name.required' => 'A name is required.',
        ];
    }
}

// Controller
class UserController extends Controller
{
    public function store(CreateUserRequest $request)
    {
        // $request is already validated
        $user = User::create($request->validated());
        return response()->json($user, 201);
    }
}
```

### OpenAPI

Laravel uses packages like `l5-swagger` or `scramble` (which auto-generates from code). `scramble` is notable because it infers schemas from Eloquent models and Form Request rules — similar to FastAPI's approach but for PHP.

### Key Design Insight

Laravel's **Form Request** pattern is elegant: validation rules are a separate class that's type-hinted in the controller method. The framework automatically instantiates the Form Request, runs validation, and either proceeds (injecting the validated request) or returns errors. This keeps validation logic organized and reusable while being tied to specific endpoints.

---

## 17. Comparative Analysis <a name="comparative-analysis"></a>

### Route Registration Patterns

| Pattern | Frameworks | Pros | Cons |
|---|---|---|---|
| **Decorators/Annotations** | FastAPI, Flask, Spring, Actix | Co-located with handler; readable | Requires language support for decorators |
| **Fluent/Builder** | Express, Fastify, ASP.NET, Axum | Composable; no special syntax needed | Metadata separated from handler body |
| **DSL** | Rails, Phoenix, Laravel | High-level; convention-driven | Routes far from handler code |
| **Schema-object** | Fastify, Hono | Route is inspectable data | More verbose |

### Where Does Schema Live?

| Approach | Frameworks | OpenAPI Ready? |
|---|---|---|
| **In the route declaration** | FastAPI, Fastify, Hono, Spring | Yes — directly |
| **In separate schema classes** | Flask-Smorest, Laravel, Grape | Yes — with wiring |
| **In struct/type definitions** | Gin, Actix, Axum | Partially — needs extraction |
| **In model layer** | Rails, Phoenix | No — separate concern |
| **Nowhere (DIY)** | Express, vanilla Flask | No |

### Should the Router Validate?

The ecosystem has converged on **yes, the router/framework should validate**, with nuance:

| Level | What validates | Example |
|---|---|---|
| **Deserialization** | "Is this valid JSON? Do the types match?" | All frameworks do this |
| **Schema validation** | "Are required fields present? Are values in range?" | FastAPI, Fastify, Spring, Laravel — automatic. Gin, Express — manual. |
| **Business validation** | "Does this email already exist? Is this user allowed to do this?" | Always in the handler/service layer |

The best practice is: **the router handles structural/schema validation automatically; the handler handles business logic validation.** This separation is clean and universal.

### OpenAPI Generation Approach

| Strategy | How it works | Frameworks |
|---|---|---|
| **Automatic from code** | Route + schema metadata → spec | FastAPI, Fastify, Hono, Spring, ASP.NET |
| **Comment/annotation scanning** | Special comments parsed by a tool | Gin (swaggo), Express (swagger-jsdoc) |
| **Test-driven** | Specs generated from integration tests | rspec-openapi (Rails) |
| **Manual** | Hand-write the spec | Any framework |

The first approach (automatic from code) is overwhelmingly preferred and is what you should target for Jarvis.

---


## 18. Ideas for Dyalog Jarvis <a name="recommendations"></a>

This section explores various possibilities for a Jarvis router, drawing on patterns from the frameworks surveyed above. These are ideas to mix, match, and pick from — not a single prescriptive design.

The guiding principle is **one function, one endpoint**. An APLer writes a function. That function becomes an endpoint. Everything else — schemas, validation, documentation, OpenAPI — is progressive enhancement layered on top.

A key enabler throughout is **APLAN (APL Array Notation)**, which lets you express nested data structures cleanly. APLAN makes the "route-as-data" patterns (Ideas B, F) dramatically more readable than traditional nested enclosed pairs.

To make comparison easy, every idea below uses the **same endpoint**:

- **POST /users** — Create a new user
- **Body:** `name` (string, required, max 100 chars), `email` (string, required, email format), `age` (integer, optional, 0–150)
- **Response (201):** `id` (integer), `name` (string), `email` (string)
- **Summary:** "Create a new user"
- **Tags:** `users`

And the handler function is always:

```apl
∇ r←CreateUser req
  r←(
	 id: 1
	 name: req.name
	 email: req.email
  )
∇
```

The **schemas** referenced throughout are defined once and reused. In APLAN they read almost like JSON Schema itself:

```apl
CreateUserBody←(
    type: 'object'
    required: 'name' 'email'
    properties: (
        name:  (type: 'string' ⋄ minLength: 1 ⋄ maxLength: 100 ⋄ description: 'Full name')
        email: (type: 'string' ⋄ format: 'email')
        age:   (type: 'integer' ⋄ minimum: 0 ⋄ maximum: 150)
    )
)

CreateUserResponse←(
    type: 'object'
    properties: (
        id:    (type: 'integer')
        name:  (type: 'string')
        email: (type: 'string')
    )
)
```

These convert to JSON trivially via `⎕JSON` and map 1:1 to OpenAPI's schema format.

---

### Idea A: Minimal Positional Registration (Express-style)

The simplest possible thing. Method is the function name on the router, path is left, handler name is right.

```apl
router.Get '/users/{id}' 'GetUser'
router.Post '/users' 'CreateUser'
router.Delete '/users/{id}' 'DeleteUser'
```

An APLer who has never seen REST can read this and understand it. There's nowhere to hang metadata, but that's fine if you don't need it yet.

**Inspired by:** Express, Gin, Echo.

**Pros:** Dead simple. Zero learning curve. One line, one route.

**Cons:** No schemas, no docs, no validation info. OpenAPI generation can only produce a skeleton ("POST /users exists").

---

### Idea B: APLAN Options on Registration (Fastify-style)

The registration call takes an optional right argument — an APLAN structure carrying all metadata about the route.

```apl
⍝ Minimal — same as Idea A:
router.Post '/users' 'CreateUser'

⍝ With full metadata:
router.Post '/users' 'CreateUser' (
    summary: 'Create a new user'
    description: 'Creates a user and returns the created resource.'
    tags: ('users'⋄)
    body: CreateUserBody
    response: (201 CreateUserResponse)
    errors: (
        (422 'Validation error')
        (409 'Email already exists')
    )
)
```

Or with the schema inlined directly (no separate variable needed):

```apl
router.Post '/users' 'CreateUser' (
    summary: 'Create a new user'
    tags: ('users'⋄)
    body: (
        type: 'object'
        required: 'name' 'email'
        properties: (
            name:  (type: 'string' ⋄ minLength: 1 ⋄ maxLength: 100 ⋄ description: 'Full name')
            email: (type: 'string' ⋄ format: 'email')
            age:   (type: 'integer' ⋄ minimum: 0 ⋄ maximum: 150)
        )
    )
    response: (
        type: 'object'
        properties: (
            id:    (type: 'integer')
            name:  (type: 'string')
            email: (type: 'string')
        )
    )
)
```

This is the Fastify model — the route registration carries everything — but APLAN makes it look almost like YAML. Compare this with Fastify's actual JavaScript:

```javascript
// Fastify (JavaScript) — same endpoint:
fastify.post('/users', {
    schema: {
        summary: 'Create a new user',
        tags: ['users'],
        body: {
            type: 'object',
            required: ['name', 'email'],
            properties: {
                name:  { type: 'string', minLength: 1, maxLength: 100 },
                email: { type: 'string', format: 'email' },
                age:   { type: 'integer', minimum: 0, maximum: 150 },
            }
        }
    },
    handler: async (request, reply) => { ... }
});
```

The APLAN version is just as readable, arguably more so.

**Inspired by:** Fastify (JSON Schema on every route), Hono (route-as-data-object).

**Pros:** All metadata in one place. Route is inspectable data. JSON Schema maps directly to OpenAPI. Progressive — omit the third argument for simple routes. APLAN makes it natural to read and write.

**Cons:** Large inline schemas make the registration call long (mitigated by defining schemas as separate variables). An APLer unfamiliar with JSON Schema still needs to learn its vocabulary.

---

### Idea C: Builder / Chaining (ASP.NET Minimal API-style)

`router.Post` returns a route object. You chain further calls to attach metadata incrementally.

```apl
route←router.Post '/users' 'CreateUser'
route.Summary 'Create a new user'
route.Tags 'users'
route.Body CreateUserBody
route.Response 201 CreateUserResponse
route.Error 422 'Validation error'
```

**Inspired by:** ASP.NET Core Minimal APIs (`.WithSummary()`, `.Produces<T>()`), Grape's block DSL.

**Pros:** Very readable — each piece of metadata is a named call. Easy to add one thing at a time. Self-documenting.

**Cons:** Requires the router to return route objects with methods. More moving parts than a plain data structure. Multiple statements per route instead of one. The route object is mutable state, which is less APL-like.

---

### Idea D: Convention-Based Auto-Registration (Rails-style)

The router scans the workspace for functions matching a naming pattern and registers them automatically.

```apl
⍝ These functions exist in the workspace:
⍝   GET∆users         → GET  /users
⍝   POST∆users        → POST /users
⍝   GET∆users∆{id}    → GET  /users/{id}
⍝   DELETE∆users∆{id}  → DELETE /users/{id}

router.AutoRegister ⍬
```

So the "Create user" handler would be named:

```apl
∇ r←POST∆users req
  r←⎕NS ⍬
  r.id←1
  r.name←req.name
  r.email←req.email
∇
```

The router calls `⎕NL` to find all functions matching the pattern, parses each name to extract the method and path segments, and registers them.

Schema metadata could live in a companion variable following a naming convention:

```apl
⍝ The router looks for POST∆users∆META alongside POST∆users:
POST∆users∆META←(
    summary: 'Create a new user'
    tags: ('users'⋄)
    body: CreateUserBody
    response: (201 CreateUserResponse)
)
```

**Inspired by:** Rails (`resources :users`), Next.js file-based routing, PHP frameworks with naming conventions.

**Pros:** Zero registration boilerplate. Add a function, it becomes an endpoint. Very "APL" in feel — the workspace *is* the API. The companion variable pattern keeps metadata co-located.

**Cons:** Path parameters are awkward to encode in names. Deeply nested paths get ugly, and how do you add parameters (`GET∆api∆v1∆users∆⍙id∆orders`). The naming convention is an implicit contract that's easy to break.

---

### Idea E: Structured Comments in the Handler (Go swaggo-style)

Metadata lives inside the handler function as structured comments. The router reads function source via `⎕NR` and extracts it.

```apl
∇ r←CreateUser req
 ⍝@POST /users
 ⍝@summary Create a new user
 ⍝@tags users
 ⍝@body CreateUserBody
 ⍝@response 201 CreateUserResponse
 ⍝@error 422 Validation error
  r←⎕NS ⍬
  r.id←1
  r.name←req.name
  r.email←req.email
∇
```

Registration is just:

```apl
router.Register 'CreateUser'
⍝ or even: router.AutoRegister ⍬  — scans all fns for ⍝@ comments
```

**Inspired by:** Go's `swaggo/swag` (doc comment scanning), Java's Javadoc, JSDoc + `swagger-jsdoc`.

**Pros:** True "one function, one endpoint" — everything about the route is inside the function. No separate registration table to keep in sync. Very greppable.

**Cons:** Comments aren't executable code — they can drift from reality and there's no check that referenced schema names actually exist. The router needs a comment parser. Limited expressiveness — complex schemas can't be inlined in comments, so you still need the APLAN schema definitions from above. Refactoring tools won't update comments when you rename a schema.

---

### Idea F: Namespace-Per-Endpoint (Spring Controller-style)

Each endpoint is a namespace containing a handler function and metadata fields. With APLAN, the metadata is clean:

```apl
:Namespace CreateUser
    Method←'POST'
    Path←'/users'

    Meta←(
        summary: 'Create a new user'
        description: 'Creates a user and returns the created resource.'
        tags: ('users'⋄)
        body: (
            type: 'object'
            required: 'name' 'email'
            properties: (
                name:  (type: 'string' ⋄ minLength: 1 ⋄ maxLength: 100 ⋄ description: 'Full name')
                email: (type: 'string' ⋄ format: 'email')
                age:   (type: 'integer' ⋄ minimum: 0 ⋄ maximum: 150)
            )
        )
        response: (
            type: 'object'
            properties: (
                id:    (type: 'integer')
                name:  (type: 'string')
                email: (type: 'string')
            )
        )
    )

    ∇ r←Handle req
      r←⎕NS ⍬
      r.id←1
      r.name←req.name
      r.email←req.email
    ∇
:EndNamespace
```

Registration scans for namespaces with the required fields:

```apl
router.RegisterNamespace CreateUser
⍝ or auto-discover all namespaces containing a Method and Path field
```

**Inspired by:** Spring Boot (`@RestController` classes), Flask `MethodView`, .NET Controllers.

**Pros:** Everything co-located. Metadata is real executable APL (variables, not comments). Easy to inspect programmatically — `ns.Method`, `ns.Meta`, etc. The namespace is a self-contained, portable unit.

**Cons:** Heavier than "one function, one endpoint" — each endpoint is a namespace with a handler inside it, not just a function. More ceremony for simple endpoints. May feel like overkill to an APLer who just wants to expose a function.

---

### Idea G: Schema Helper Functions (Simplified DSL)

Instead of writing JSON Schema directly (even in APLAN), provide helper functions that produce JSON Schema under the hood. This is for APLers who don't want to learn JSON Schema vocabulary at all.

```apl
⍝ Define fields
fName←String 'name' (required: 1 ⋄ maxLength: 100 ⋄ description: 'Full name')
fEmail←String 'email' (required: 1 ⋄ format: 'email')
fAge←Integer 'age' (minimum: 0 ⋄ maximum: 150)
fId←Integer 'id' ⍬

⍝ Compose into schemas
CreateUserBody←Schema fName fEmail fAge
CreateUserResponse←Schema fId fName fEmail
```

`String`, `Integer`, `Number`, `Boolean`, `Array` are framework-provided field constructors. `Schema` combines fields into a JSON Schema object. The output is identical to writing the APLAN schema by hand — the helpers just provide a friendlier entry point.

Compare all three ways to define the same schema:

```apl
⍝ 1. Raw JSON Schema in APLAN (Idea B) — most explicit, 1:1 with OpenAPI:
CreateUserBody←(
    type: 'object'
    required: 'name' 'email'
    properties: (
        name:  (type: 'string' ⋄ maxLength: 100)
        email: (type: 'string' ⋄ format: 'email')
        age:   (type: 'integer' ⋄ minimum: 0 ⋄ maximum: 150)
    )
)

⍝ 2. Helper DSL (Idea G) — friendlier, same output:
CreateUserBody←Schema (String 'name' (required:1⋄maxLength:100)) (String 'email' (required:1⋄format:'email')) (Integer 'age' (minimum:0⋄maximum:150))

⍝ 3. Example-based (Idea H) — most intuitive, least precise:
CreateUserBody←InferSchema (name: 'Jane' ⋄ email: 'j@x.com' ⋄ age: 30)
```

**Inspired by:** Marshmallow (Python), Joi/Zod (Node.js), Laravel's validation rules.

**Pros:** No JSON Schema knowledge needed. Composable — fields are reusable values. Type-specific constructors (`String`, `Integer`) prevent mismatches. Still produces JSON Schema under the hood.

**Cons:** Another abstraction layer. Some JSON Schema features (`oneOf`, `allOf`, conditional schemas) may be hard to express. An APLer will eventually need to understand JSON Schema anyway when debugging OpenAPI output.

---

### Idea H: Schema-by-Example with Refinements

Start from an example value. The framework infers types. Then optionally refine with constraints.

```apl
⍝ Start from an example — types are inferred automatically
CreateUserBody←InferSchema (name: 'Jane Doe' ⋄ email: 'jane@example.com' ⋄ age: 30)

⍝ Refine: mark fields required, add constraints
CreateUserBody←CreateUserBody Require 'name' 'email'
CreateUserBody←CreateUserBody Constrain 'name' (maxLength: 100)
CreateUserBody←CreateUserBody Constrain 'email' (format: 'email')
CreateUserBody←CreateUserBody Constrain 'age' (minimum: 0 ⋄ maximum: 150)
```

Without any refinements, `InferSchema` alone gives you a working schema with correct types — good enough for basic validation and a useful OpenAPI spec.

**Inspired by:** TypeBox (TypeScript), FastAPI's type inference from Python hints, schema-by-example tools.

**Pros:** "Show me what the data looks like" is the most intuitive starting point. No JSON Schema vocabulary needed upfront. Refinements are optional — the base example alone is useful. Very APL-like in philosophy: work from concrete data, not abstract types.

**Cons:** The example must be representative (if `age` is omitted, it won't appear in the schema). Refinement calls add up for complex schemas. No way to express optional fields vs. missing fields in the example alone.

---

### Cross-Cutting Ideas (applicable to any registration approach above)

#### Should the Router Validate?

The survey shows strong consensus: **yes**, with nuance.

| Level | What it does | Who does this |
|---|---|---|
| **Level 0** | No validation. Body passed through raw. | Express (vanilla), Gin (if you forget) |
| **Level 1** | Type coercion — path param `{id}` becomes a number | Most frameworks, at minimum |
| **Level 2** | Full schema validation before handler. 422 on failure. | FastAPI, Fastify, Spring, Laravel |
| **Level 3** | Also validate handler's *response* against schema (dev mode) | Fastify |

A pragmatic approach for Jarvis: **if a route has a schema, validate against it; if it doesn't, pass data through raw.** Simple routes have zero overhead; schema-rich routes get automatic validation. The handler never needs to check "is `name` present?" — only "does this email already exist in the database?"

#### Path Parameter Styles

| Style        | Example           | Used by                                    |
| ------------ | ----------------- | ------------------------------------------ |
| Curly braces | `/users/{id}`     | OpenAPI, FastAPI, Spring, Jarvis (current) |
| Colon prefix | `/users/:id`      | Express, Gin, Echo, Phoenix                |
| Typed        | `/users/{id:int}` | ASP.NET, Flask                             |

Jarvis already uses `{id}` and that matches OpenAPI natively. Adding an optional type suffix (`{id:int}`) could enable automatic coercion and feed the type into the OpenAPI spec without a separate parameter schema.

#### Route Grouping and Prefixes

```apl
api←router.Group '/api/v1'
api.Get '/users/{id}' 'GetUser'
api.Post '/users' 'CreateUser'

⍝ Group with shared middleware
admin←(router.Group '/admin').Use 'AuthRequired'
admin.Delete '/users/{id}' 'DeleteUser'
```

#### Handler Function Contract

What does a handler receive? A few possibilities that work with any registration approach:

```apl
⍝ Option 1: Full request namespace — most flexible
∇ r←CreateUser req
  ⍝ req.body req.params req.query req.headers all available
  r←⎕NS ⍬
  r.id←1
  r.name←req.body.name
∇

⍝ Option 2: Just the body — simplest for POST/PUT
∇ r←CreateUser body
  ⍝ body is already validated, just the JSON payload as a namespace
  r←⎕NS ⍬
  r.id←1
  r.name←body.name
∇

⍝ Option 3: Dyadic for routes with path params + body
∇ r←id UpdateUser body
  ⍝ left arg = path param(s), right arg = body
  r←⎕NS ⍬
  r.id←id
  r.name←body.name
∇

⍝ Option 4: Niladic for no-input routes
∇ r←HealthCheck
  r←⎕NS ⍬
  r.status←'ok'
∇
```

The router could inspect the function header (via `⎕ATX` or `⎕NR`) to determine arity and pass arguments accordingly.

#### OpenAPI Generation

If routes carry metadata, generation is a walk over the route registry:

```apl
spec←router.GenerateOpenAPI (title: 'My API' ⋄ version: '1.0.0')
⍝ spec is a JSON string conforming to OpenAPI 3.0
```

How much metadata determines how rich the spec is:

| What you provide | What the spec contains |
|---|---|
| Method + path only | Skeleton — "POST /users exists" |
| + body & response schemas | Input/output documentation |
| + summary, description, tags | Production-quality docs with Swagger UI grouping |
| + examples, error responses | Full reference documentation |

Since APLAN schemas are structurally identical to JSON Schema, the translation to OpenAPI's `components/schemas` is nearly trivial — it's mostly a matter of wrapping the route registry into the OpenAPI envelope.

#### Error Handling

```apl
⍝ Option 1: Signal — router catches and maps to HTTP status
∇ r←GetUser id
  :If 0=≢Users.Lookup id
    ⎕SIGNAL⊂('EN' 404)('Message' 'User not found')
  :EndIf
  r←Users.Lookup id
∇

⍝ Option 2: Return a structured error — router inspects the result
∇ r←GetUser id
  :If 0=≢Users.Lookup id
    r←⎕NS ⍬ ⋄ r.error←1 ⋄ r.status←404 ⋄ r.message←'User not found'
    :Return
  :EndIf
  r←Users.Lookup id
∇
```

Both approaches can coexist — the router `:Trap`s signals *and* checks for an `error` field in the result.

#### Middleware / Hooks

```apl
router.Use 'LogRequest'              ⍝ global — runs on every request
router.OnError 'HandleError'         ⍝ global error handler

⍝ per-route (using APLAN options from Idea B):
router.Post '/users' 'CreateUser' (
    middleware: 'AuthRequired' 'RateLimit'
    summary: 'Create a new user'
    body: CreateUserBody
)
```

---

### The Key Principle: Progressive Enhancement

The absolute minimum should be:

```apl
router.Post '/users' 'CreateUser'
```

One line. The handler is a normal APL function that takes data and returns data. No schemas, no metadata, no ceremony. It just works.

Later, the APLer adds schemas, docs, and validation *without changing the handler function*:

```apl
router.Post '/users' 'CreateUser' (
    summary: 'Create a new user'
    tags: ('users'⋄)
    body: CreateUserBody
)
```

The handler didn't change. The router just knows more about the endpoint now. Start simple, add richness as needed.