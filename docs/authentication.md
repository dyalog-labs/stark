# Authentication

Stark ties authentication to OpenAPI [security schemes](https://spec.openapis.org/oas/v3.0.3#security-scheme-object). You register each scheme together with a hook function. Stark then:

- adds the scheme to `components.securitySchemes` in the generated spec
- enforces each route's `security` requirement before calling its handler

The spec and the server's behaviour come from the same declarations, so they can't drift apart.

## Quick start

```apl
router.AddSecurity 'bearerAuth' (type:'http' ⋄ scheme:'bearer') 'CheckBearer'
router.Spec.security←,(bearerAuth:⍬)                ⍝ every route needs a token...

'/health' router.Get ('Health' (security:⍬))         ⍝ ...except this one
'/me'     router.Get 'Me'

∇ user←auth CheckBearer req
  user←LookupToken auth.credential    ⍝ the user, or 0 to reject
∇

∇ result←Me req
  result←req.User                     ⍝ whatever CheckBearer returned
∇
```

A request to `/me` without a valid token gets `401` with `WWW-Authenticate: Bearer`. With a valid token, the handler runs and `req.User` holds the user.

## Registering a scheme

```apl
router.AddSecurity name definition hook
```

| Argument     | Description |
|--------------|-------------|
| `name`       | The scheme name used in `security` requirements, e.g. `'bearerAuth'` |
| `definition` | An OpenAPI Security Scheme Object, written into the spec as-is |
| `hook`       | Name of a dyadic result-returning function in `Handlers` that checks the credential |

Registering the same name twice signals EN 11. You don't need to write `components.securitySchemes` in `router.Spec` yourself; any schemes you do put there are kept alongside the registered ones.

## Which routes are protected

Each route's security requirement is, in order of precedence:

1. the route's own `security` option, if present (`security:⍬` makes the route public)
2. otherwise `router.Spec.security`, the global default
3. otherwise nothing: the route is public

| Route options    | `Spec.security` set    | `Spec.security` not set |
|------------------|------------------------|-------------------------|
| no `security`    | uses the global default | public                 |
| `security:⍬`     | public                 | public                  |
| `security:(…)`   | uses the route's own rule | uses the route's own rule |

This matches OpenAPI, so the spec always describes what the server enforces. Once a global default is set, a route you forget to mark stays protected.

## Requirement syntax

A security requirement is a vector of namespaces. Each namespace maps scheme names to the scopes they require:

```apl
,(bearerAuth:⍬)                        ⍝ bearer token required
(bearerAuth:⍬)(apiKey:⍬)               ⍝ bearer token OR api key
,(bearerAuth:⍬ ⋄ apiKey:⍬)             ⍝ bearer token AND api key
,(oauth:,⊂'admin')                     ⍝ oauth2 token with the 'admin' scope
(bearerAuth:⍬)()                       ⍝ bearer token optional: () lets anyone through
```

The entries are alternatives: Stark tries them in order and the first one that passes wins. Every scheme inside an entry must pass.

!!! note
    OpenAPI 3.0.3 only allows scopes for `oauth2` and `openIdConnect` schemes. For `http` and `apiKey` schemes, the list must be `⍬`. Stark prints a warning at `Start` if a route gives scopes to another scheme type.

## Writing a hook

The hook is called as:

```apl
user←auth Hook req
```

`auth` is a namespace with:

| Member       | Description |
|--------------|-------------|
| `scheme`     | The scheme name, e.g. `'bearerAuth'` |
| `credential` | The credential Stark extracted from the request (see below) |
| `scopes`     | The scopes the route requires, as a vector of strings (`⍬` if none) |

The hook returns:

- **the authenticated user** — any value except the scalar `0`. Stark stores it in `req.User`.
- **`0`** to reject the request. Stark responds `401 Unauthorized`.

To reject a user who is authenticated but not allowed, call `req.Fail 403` and return `0`. Stark keeps the 403:

```apl
∇ user←auth CheckOAuth req
  user←LookupToken auth.credential
  :If 0≢user
  :AndIf ~∧/auth.scopes∊user.scopes
      req.Fail 403 ⋄ user←0           ⍝ authenticated but missing a scope
  :EndIf
∇
```

If an error occurs in a hook, it is handled like an error in a handler: it goes to [`OnErrorFn`](stark-router.md#error-handling) if set, and is re-signalled otherwise.

## Credentials

Stark extracts the credential from the request according to the scheme's definition:

| Scheme definition | `auth.credential` |
|-------------------|-------------------|
| `type:'http' ⋄ scheme:'bearer'` | The token after `Bearer ` in the `Authorization` header |
| `type:'http' ⋄ scheme:'basic'`  | `(userid password)`, decoded from the `Authorization` header |
| `type:'oauth2'` or `type:'openIdConnect'` | The token after `Bearer ` in the `Authorization` header |
| `type:'apiKey' ⋄ in:'header' ⋄ name:'X-API-Key'` | The value of that header |
| `type:'apiKey' ⋄ in:'query' ⋄ name:'api_key'`    | The value of that query parameter |
| `type:'apiKey' ⋄ in:'cookie' ⋄ name:'session'`   | The value of that cookie |
| any other type | `⍬`; the hook is always called and reads `req` itself |

If the request doesn't carry the credential, Stark treats that scheme as failed without calling the hook.

## What the handler sees

| Member         | Description |
|----------------|-------------|
| `req.User`     | The user returned by the hook. For an entry with several schemes, the user from the first scheme in alphabetical order. `⍬` on public routes. |
| `req.Security` | A namespace mapping each scheme that passed to its user, e.g. `req.Security.bearerAuth`. Empty on public routes. |

## Failed requests

When no alternative passes, Stark responds with:

- `401` and `(detail: 'Not authenticated')`, plus a `WWW-Authenticate` header for each `http`, `oauth2` or `openIdConnect` scheme the route accepts: `Bearer`, or `Basic realm="<Info.title>", charset="UTF-8"`
- or, if a hook called `req.Fail` with another error status such as 403, that status and `(detail: 'Forbidden')` (the standard status text)

If one alternative fails with a 403 but a later one passes, the 403 is undone.

## Protecting /openapi.json

The spec endpoint is public by default, even when a global default is set. This follows FastAPI and Fastify. To protect it, set `DocsSecurity`:

```apl
router.DocsSecurity←,(bearerAuth:⍬)
```

## Checks at Start

`Start` signals EN 11 if:

- a route, `Spec.security` or `DocsSecurity` names a scheme that wasn't registered with `AddSecurity`
- a hook doesn't exist in `Handlers`, or isn't a result-returning dyadic function

## Debugging

With `router.Debug←32`, Stark prints each scheme it checks and the outcome:

```
STARK: GET /me → Me
STARK: auth bearerAuth → ok
```

## Jarvis's AuthenticateFn

Leave Jarvis's own `AuthenticateFn` unset when using Stark. It applies to every request, including `/openapi.json`, and can't tell 401 from 403.
