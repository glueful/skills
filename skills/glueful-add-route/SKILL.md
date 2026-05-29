---
name: glueful-add-route
description: Register or modify routes in a Glueful framework project — the fluent router (get/post/put/delete, groups, where constraints), attribute routing, middleware, rate limiting, and the OpenAPI docblock annotations (@route/@summary/@requestBody/@response) that generate the API spec/SDK. Use when adding or documenting API endpoints in a Glueful app. Glueful routing is NOT Laravel — there is no Route facade; route files receive a $router instance, rate limits must use the builder method (not a middleware string), and docs come from route docblocks.
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

String middleware names resolve through the **container** (`$container->get($name)`) — so `'auth'`, `'rate_limit'`, etc. are container service IDs, and a custom middleware must be registered there to be referenced by name. Middleware implements `Glueful\Routing\Middleware\RouteMiddleware`:

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

## OpenAPI doc annotations

Glueful generates its OpenAPI spec from **docblock annotations above each route** (parsed by `CommentsDocGenerator`). Build the spec with `php glueful generate:openapi`, then generate an SDK — `generate:client` takes a **required language argument**:

```bash
php glueful generate:openapi
php glueful generate:client typescript --spec openapi.json -o ./generated   # language is required
# languages: typescript|ts, python, go, ruby, java, … ; --spec defaults to openapi.json, -o to ./generated
```

Annotate routes as you add them — the generated docs/SDK are only as good as these blocks.

### The tags (exact grammar)

```php
/**
 * @route POST /articles/{id}/comments
 * @summary Add Comment
 * @description Adds a comment to an article.
 * @tag Comments
 * @requiresAuth true
 * @requestBody body:string="Comment text" parent_id:string="Parent comment UUID" {required=body}
 * @response 201 application/json "Comment created" {
 *   success:boolean="true",
 *   message:string="Success message",
 *   data:{
 *     id:string="Comment UUID",
 *     body:string="Comment text",
 *     created_at:string="ISO timestamp"
 *   }
 * }
 * @response 401 "Authentication required"
 * @response 422 "Validation failed"
 */
$router->post('/articles/{id}/comments', [CommentController::class, 'store'])
    ->where('id', '\d+')->middleware(['auth']);
```

- **`@route METHOD /path`** — required; the parser keys off this (regex `@route\s+([A-Z]+)\s+(\S+)`). No `@route`, no operation.
- **`@summary` / `@description` / `@tag`** — free text; `@tag` groups operations in the spec (use a consistent tag like `Articles`).
- **`@requiresAuth true`** — marks the operation as secured.
- **`@requestBody`** — space-separated `field:type="description"` tokens; enums as `field:type[a,b,c]="..."`; mark required with a trailing `{required=field1,field2}`. May span multiple lines.
- **`@response CODE [contentType] "description" { schema }`** — `contentType` defaults to `application/json`; the optional `{ schema }` uses the same `field:type="desc"` syntax and nests with `field:{ ... }`. Omit the schema for simple statuses (`@response 401 "Unauthorized"`). Document the `{success, message, data}` envelope shape your controller returns.

### Parameters

- **Path params auto-derive from `{param}` — but only when there are *no* `@param` tags at all.** The generator derives path params from the route path only if you wrote zero `@param` lines. The moment you add any `@param` (e.g. a query param), auto-derivation switches off and you must declare **every** param explicitly, including the path ones:
  ```
  @param id   path  string  true   "Article ID"
  @param page query integer false  "Page number"
  ```
  So: a route with just `{id}` and no `@param` → `id` is documented automatically; a route with `{id}` *and* a `@param page query ...` → you must also add `@param id path string true "..."` or `id` goes undocumented.
- **`@param` grammar is space-separated:** `@param NAME (path|query|header|cookie) TYPE (true|false) "description"` — never `name:type=` (that breaks PHPDoc tooling, see gotchas).
- **operationId is generated for you** (`OperationIdGenerator` makes a unique camelCase id from the method + path) — don't hand-write one.

### Gotchas (these have actually bitten)

- **Never write `@param uuid:string="..."`.** That DSL doesn't match the generator's `@param` grammar *and* `@param` is a real PHPDoc tag, so IDEs/intelephense raise parse errors (P1129/P1133). Use `{param}` in the path (auto-derived) or the space-separated `@param` form above.
- **File-level docblock: fully-qualify the `@var`** — `@var \Glueful\Routing\Router $router` (short `Router` trips PHPDoc tooling when the file has no `Router` usage in code).
- **A route without `@route` is invisible** to the generator — every documented endpoint needs it.

## Not-Laravel reminders

- No `Route::get(...)` facade → `$router->get(...)` inside a route file.
- No `->middleware('throttle:60,1')` for rate limits → `->rateLimit(60, 1)` builder + attach `'rate_limit'`.
- No `Route::resource(...)` → register verbs explicitly, or use `#[Controller]` + verb attributes.
- Route model binding is not automatic → resolve the entity in the controller from the typed `{id}` argument.

## Checklist

- [ ] Registered on `$router` in a `routes/*.php` file (added to `RouteManifest` if it's a framework route), or via `#[Controller]`/verb attributes.
- [ ] Rate limits use `->rateLimit($attempts, $perMinutes)` (builder), not the `rate_limit:N,W` string form.
- [ ] Named middleware exists as a container service; custom middleware implements `RouteMiddleware`.
- [ ] Path-param constraints via `->where(...)`; route named via `->name(...)`.
- [ ] Optional/feature-flagged routes guarded with an `env()` early return.
- [ ] Documented with `@route`/`@summary`/`@tag`/`@requestBody`/`@response` (envelope schema); `operationId` left to the generator. Path params auto-derive from `{param}` **only when there are no `@param` tags** — if any `@param` is present, declare path params explicitly too (`@param id path string true "…"`).
- [ ] No malformed `@param name:type=...` lines; file-level `@var \Glueful\Routing\Router $router` fully qualified. Ran `generate:openapi` to confirm the endpoint appears.
