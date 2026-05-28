---
name: glueful-add-route
description: Register or modify routes in a Glueful framework project — the fluent router (get/post/put/delete, groups, where constraints), attribute routing, middleware, and rate limiting. Use when adding API endpoints' routing in a Glueful app. Glueful routing is NOT Laravel — there is no Route:: facade; route files receive a $router instance, and rate limits must use the builder method, not a middleware string.
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

## OpenAPI doc annotations (optional but encouraged)

Framework route files document endpoints with `@route`/`@summary`/`@requestBody`/`@response` docblock annotations above each route (parsed by the docs generator). Path params are auto-derived from `{param}` in the route path — don't add a `@param name:type=...` line (it doesn't match the generator's format and trips PHPDoc tooling). In the file-level docblock, fully-qualify `@var \Glueful\Routing\Router $router`.

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
- [ ] (If a framework route) file-level `@var \Glueful\Routing\Router $router` is fully qualified; no malformed `@param` lines.
