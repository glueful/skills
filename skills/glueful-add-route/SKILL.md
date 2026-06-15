---
name: glueful-add-route
description: Register or modify routes in a Glueful framework project — the fluent router (get/post/put/delete, groups, where constraints), attribute routing, middleware, rate limiting, and OpenAPI documentation via the reflect generator (typed #[ApiOperation]/#[QueryParam]/#[ApiRequestBody]/#[ApiResponse] attributes on the controller method, plus typed RequestData/ResponseData DTOs). Use when adding or documenting API endpoints in a Glueful app. Glueful routing is NOT Laravel — there is no Route facade; route files receive a $router instance, rate limits must use the builder method (not a middleware string), and (framework ≥ 1.57.0) OpenAPI docs come from typed attributes/types, NOT route docblocks.
---

# Adding a Glueful Route

Routes are registered against a `$router` instance (`Glueful\Routing\Router`), not a `Route::` facade. Route files live in `routes/*.php` and are loaded via `RouteManifest`.

## Route files & the manifest

```php
<?php
/** @var \Glueful\Routing\Router $router */   // $router is injected by RouteManifest

use Glueful\Routing\Router;
use App\Controllers\WidgetController;

$router->group(['prefix' => '/widgets', 'middleware' => ['auth']], function (Router $router) {
    $router->get('/', [WidgetController::class, 'index'])->name('widgets.index');
    $router->get('/{id}', [WidgetController::class, 'show'])->where('id', '\d+')->name('widgets.show');
    $router->post('/', [WidgetController::class, 'store'])->name('widgets.store');
});
```

Framework route files are registered in `RouteManifest` (`src/Routing/RouteManifest.php`):
- **`api_routes`** — get the configured API prefix (e.g. `/api/v1/...`).
- **`public_routes`** — no API prefix (health, docs).
- **`app_routes_dir`** — the consumer app's `routes/` directory; all `*.php` there is auto-discovered (files prefixed `_` are treated as partials and skipped).

A new framework route file must be added to the appropriate manifest list; a new **app** route file just needs to land in the app's `routes/` directory.

## Verb helpers & fluent builder

`$router->get|post|put|delete(string $path, mixed $handler): Route`. The handler is `[Controller::class, 'method']`. Chainable on the returned `Route`:

- `->where(string|array $param, ?string $regex = null)` — path-param constraints.
- `->middleware(string|array $middleware)` — named middleware (resolved via the container) or class/closure.
- `->name(string $name)` — route name.
- `->rateLimit(int $attempts, int $perMinutes = 1, ?string $tier = null, string $algorithm = 'sliding', string $by = 'ip')` — see below.

## Rate limiting — use the builder, NOT the string form

This is the most common Glueful routing mistake. `EnhancedRateLimiterMiddleware` reads its limits from `Route::getRateLimitConfig()` — which is populated **only** by the `->rateLimit()` builder method. The `rate_limit:5,60` middleware-string form is parsed and passed to `handle()`, but the rate limiter ignores those params, so it silently does **not** enforce.

```php
// ✅ CORRECT — limits actually enforced (unit is MINUTES)
$router->post('/login', [AuthController::class, 'login'])
    ->rateLimit(5, 1)                  // 5 attempts per 1 minute
    ->middleware(['rate_limit']);      // attach the middleware so it reads the config

// ❌ WRONG — the string params are ignored; no enforcement
$router->post('/login', [AuthController::class, 'login'])
    ->middleware('rate_limit:5,60');
```

`->rateLimit()` also takes `$by` (`'ip' | 'user' | 'endpoint'`, default `'ip'`) to choose the throttle key.

## Middleware

String middleware names resolve through the **container** (`$container->get($name)`) — so `'auth'`, `'rate_limit'`, etc. are container service IDs, and a custom middleware must be registered there to be referenced by name. Middleware implements `Glueful\Routing\RouteMiddleware`:

```php
public function handle(Request $request, callable $next, ...$params): Response
```

The `name:param1,param2` syntax passes `$params` to `handle()` (works for middleware that read params — just not for rate limiting, which reads route config instead). Group middleware is inherited by all routes in the group and merges with per-route middleware.

## Conditional registration (feature flags)

Route files run plain PHP, so guard optional routes with an early return. `routes/*.php` don't have `ApplicationContext` in scope — read env directly (`env()` casts `'true'`/`'false'` to real booleans):

```php
if (env('FEATURE_X_ENABLED', false) !== true) {
    return;   // routes not registered at all → /feature-x/* returns 404 when off
}

$router->post('/feature-x/run', [FeatureXController::class, 'run'])->middleware(['auth']);
```

## Attribute routing (alternative)

Define routes on the controller with attributes from `Glueful\Routing\Attributes`:

```php
use Glueful\Routing\Attributes\{Controller, Get, Post, Fields, RequireScope};

#[Controller(prefix: '/widgets')]
final class WidgetController extends BaseController
{
    #[Get('/{id}', where: ['id' => '\d+'])]
    public function show(Request $request, int $id): Response { /* ... */ }

    #[Post('/')]
    #[RequireScope('write:widgets')]   // auto-attaches the require_scope middleware
    public function store(Request $request): Response { /* ... */ }
}
```

`#[Fields(allowed: [...], strict: true)]` declares field-selection whitelists; `#[RequireScope(...)]` is repeatable and gates by API-key scope.

## OpenAPI docs — the reflect generator (framework ≥ 1.57.0)

As of framework **1.57.0** the OpenAPI spec is built by the **reflect generator** from the **live route table + PHP types + typed attributes on the controller method** — *not* from route docblocks. The old docblock tags (`@route`, `@summary`, `@requestBody`, `@response`, `@queryParam`, the space-separated `@param`) and the `CommentsDocGenerator` were **removed**: any such blocks are now inert and silently ignored, and the `documentation.generator` / `API_DOCS_GENERATOR` switches no longer exist (reflect is the only generator). Build the spec and an SDK exactly as before:

```bash
php glueful generate:openapi
php glueful generate:client typescript --spec openapi.json -o ./generated   # language is required
# languages: typescript|ts, python, go, ruby, java, … ; --spec defaults to openapi.json, -o to ./generated
```

### What is inferred automatically — no annotation needed

The generator derives, with **zero** annotation: the path, HTTP method, and `{param}` path params (with the `->where()` regex as `pattern`); `security` from `auth`/`api_key` middleware; a `429` from `rate_limit`; scope prose from `require_scope`; the `fields`/`expand` query params from `#[Fields]`; the **request-body schema** from a typed `RequestData` parameter (plus an auto `422`); and the **success-response schema** from a `ResponseData` / `CollectionResponse` / `PaginatedResponse` return type. In practice **~95% of routes need no annotation at all** — the types are the docs.

> **Type-first rule:** the cleanest way to document an endpoint is to *type* it. A `RequestData` param and a `ResponseData` return need no annotation — the schema is reflected from the class. See `glueful-build-validation-dto` and `glueful-add-controller` for the typed-I/O DTOs.

### Annotate only what types can't express — on the controller METHOD

The override attributes live in `Glueful\Routing\Attributes\` and go on the **handler method** (not the route file). Every one is optional and additive — omitting it leaves the inferred value unchanged.

```php
use Glueful\Routing\Attributes\{Post, ApiOperation, QueryParam, ApiRequestBody, ApiResponse};

#[Post('/articles/{id}/comments')]
#[ApiOperation(summary: 'Add Comment', description: 'Adds a comment to an article.', tags: ['Comments'])]
#[QueryParam('preview', 'boolean', description: 'Render but do not persist')]
#[ApiRequestBody(schema: CreateCommentData::class)]          // typed DTO class → JSON body schema
#[ApiResponse(201, CommentData::class, description: 'Comment created')]
#[ApiResponse(422, description: 'Validation failed')]
public function store(Request $request, int $id): Response { /* ... */ }
```

- **`#[ApiOperation(summary, description, tags, operationId, deprecated)]`** — prose. A non-empty `summary`/`tags` replaces the value derived from the route name/path; `operationId` is otherwise generated for you (don't hand-write one).
- **`#[QueryParam(name, type='string', description='', required=false, format=null, enum=null)]`** (repeatable) — one per query parameter. `type ∈ string|integer|number|boolean`. Path params auto-derive from `{param}`; `fields`/`expand` come from `#[Fields]` — don't redeclare those.
- **`#[ApiRequestBody(...)]`** — document a body the generator can't infer from a *hydrating* `RequestData` param. `schema: SomeData::class` is a typed DTO **class** for a JSON body (never an inline JSON schema). For a non-JSON/multipart body use `inlineSchema: [...]` with a non-JSON `contentType` (e.g. `'multipart/form-data'`) — `inlineSchema` is rejected for `application/json`.
- **`#[ApiResponse(status, schema=null, description='', collection=false, envelope=true, contentType=..., body=null)]`** (repeatable) — declare a response by status. `schema` is a typed DTO class (`collection: true` wraps it as a list); `body: 'binary'|'text'` documents a file/HTML/stream response with no DTO. **4xx/error responses are never inferred — declare them here.**

Full reference (in the framework package): `docs/OPENAPI_REFLECT.md`, `docs/REQUEST_DTOS.md`, `docs/RESPONSE_DTOS.md`.

### Gotchas (these have actually bitten)

- **Docs moved from the route file to the controller method.** Pre-1.57.0 projects put `@route`/`@response` blocks above the `$router->...` call; those are now dead. Put `#[ApiOperation]`/`#[QueryParam]`/`#[ApiResponse]` on the **handler method** instead (alongside `#[Get]`/`#[Post]` if you use attribute routing).
- **A typed DTO beats an attribute.** Don't hand-write an `#[ApiRequestBody]`/`#[ApiResponse]` schema for a handler that already takes a `RequestData` param or returns a `ResponseData` — the generator already has it, and the attribute would just risk drifting from the real type.
- **No inline JSON request schema.** JSON bodies are expressed as a DTO **class** (`schema: …`); `inlineSchema` is for non-JSON only. This is deliberate — it keeps request shapes type-checked.
- **File-level docblock: fully-qualify the `@var`** — `@var \Glueful\Routing\Router $router` (short `Router` trips PHPDoc tooling when the file has no `Router` usage in code).

## Not-Laravel reminders

- No `Route::get(...)` facade → `$router->get(...)` inside a route file.
- No `->middleware('throttle:60,1')` for rate limits → `->rateLimit(60, 1)` builder + attach `'rate_limit'`.
- No `Route::resource(...)` → register verbs explicitly, or use `#[Controller]` + verb attributes.
- Route model binding is not automatic → resolve the entity in the controller from the typed `{id}` argument.
- No `@route`/`@response` docblocks for OpenAPI (framework ≥ 1.57.0) → typed attributes on the controller method + typed DTOs.

## Checklist

- [ ] Registered on `$router` in a `routes/*.php` file (added to `RouteManifest` if it's a framework route), or via `#[Controller]`/verb attributes.
- [ ] Rate limits use `->rateLimit($attempts, $perMinutes)` (builder), not the `rate_limit:N,W` string form.
- [ ] Named middleware exists as a container service; custom middleware implements `RouteMiddleware`.
- [ ] Path-param constraints via `->where(...)`; route named via `->name(...)`.
- [ ] Optional/feature-flagged routes guarded with an `env()` early return.
- [ ] OpenAPI: prefer typing the endpoint (a `RequestData` param / `ResponseData` return needs no annotation). Add `#[ApiOperation]`/`#[QueryParam]`/`#[ApiRequestBody]`/`#[ApiResponse]` on the **controller method** only for prose, query params, manual bodies, error codes, or non-JSON responses.
- [ ] No legacy `@route`/`@summary`/`@requestBody`/`@response`/`@queryParam` docblocks (inert on framework ≥ 1.57.0); file-level `@var \Glueful\Routing\Router $router` fully qualified. Ran `generate:openapi` to confirm the endpoint appears with the expected summary/params/responses.
