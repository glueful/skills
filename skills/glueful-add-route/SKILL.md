---
name: glueful-add-route
description: Register or modify routes in a Glueful framework project — the fluent router (get/post/put/patch/delete, groups, where constraints), attribute routing (Controller/Get/RateLimit/RequireScope/Fields attributes), middleware aliases, rate limiting, and OpenAPI documentation via the reflect generator (typed ApiOperation/QueryParam/ApiRequestBody/ApiResponse attributes on the controller method, plus typed RequestData/ResponseData DTOs). Use when adding or documenting API endpoints in a Glueful app. Glueful routing is NOT Laravel — there is no Route facade; route files receive a $router instance, controllers must be container-registered, and OpenAPI docs come from typed attributes/types, NOT route docblocks.
---

# Adding a Glueful Route

Routes are registered against a `$router` instance (`Glueful\Routing\Router`), not a `Route::` facade. App route files live in `routes/*.php` and are loaded by `RouteManifest`.

For extensions, route files are runtime wiring: load them from the extension provider's `boot()` method with `loadRoutesFrom()`, not from `register()`. Keep any config gate intact, but evaluate it in `boot()` so route registration follows the same lifecycle as commands, event listeners, and other extension runtime wiring. `loadRoutesFrom()` injects **only `$router`** into the file — get the context with `$context = $router->getContext();`.

## Route files & the manifest

```php
<?php
/** @var \Glueful\Routing\Router $router */   // injected by RouteManifest (app files also get $context)

use Glueful\Routing\Router;
use App\Controllers\WidgetController;

$router->group(['prefix' => '/v1/widgets', 'middleware' => ['auth']], function (Router $router) {
    $router->get('/', [WidgetController::class, 'index'])->name('widgets.index');
    $router->get('/{id}', [WidgetController::class, 'show'])->where('id', '\d+')->name('widgets.show');
    $router->post('/', [WidgetController::class, 'store'])->name('widgets.store');
});
```

`RouteManifest` (`src/Routing/RouteManifest.php`) loads:
- **`app_routes_dir`** — the app's `routes/` directory; every `*.php` there is auto-discovered (files prefixed `_` are partials and skipped). App routes are loaded **first** and are **not prefixed automatically** — the app owns its prefix (the skeleton uses a `v1` group). To share the framework's API prefix (`/api/v1` by default) use `$router->group(['prefix' => api_prefix($context)], ...)`.
- **`api_routes`** — framework route files (`auth`, `blobs`, `resource`, `extensions`, `webhooks`) mounted under the API prefix.
- **`public_routes`** — framework files with no prefix (health, docs).

A new framework route file must be added to the appropriate manifest list; a new **app** route file just needs to land in `routes/`.

**Register the controller in the container.** The router resolves `[Controller::class, 'method']` via the container with **no autowire fallback** — an unregistered controller fails with "Service … not found" when hit. In the skeleton add it to `app/Providers/AppServiceProvider::services()`: `WidgetController::class => ['autowire' => true, 'shared' => true]` (see `glueful-add-controller`).

## Verb helpers & fluent builder

`$router->get|post|put|patch|delete|head|options(string $path, mixed $handler): Route`. The handler is `[Controller::class, 'method']`. Chainable on the returned `Route`:

- `->where(string|array $param, ?string $regex = null)` — path-param constraints.
- `->middleware(string|array $middleware)` — middleware alias (container service ID), class, or `alias:param1,param2`.
- `->name(string $name)` — route name.
- `->rateLimit(int $attempts, int $perMinutes = 1, ?string $tier = null, string $algorithm = 'sliding', string $by = 'ip')` — see below.

Handler parameters are resolved by type, then name: the Symfony `Request`, a typed `RequestData` DTO, container services, then route params by name (`{id}` → `int $id`, cast to the declared scalar type).

## Rate limiting

Prefer the **builder** — the router auto-attaches the `rate_limit` middleware for any route carrying a limit (since 1.91; adding `'rate_limit'` yourself is harmless but unnecessary):

```php
$router->post('/login', [AuthController::class, 'login'])
    ->rateLimit(5, 1);                       // 5 attempts per 1 MINUTE, keyed by IP
$router->post('/export', [ExportController::class, 'run'])
    ->rateLimit(10, 60, by: 'user');         // 10 per hour per user ('ip' | 'user' | 'endpoint')
```

The string form also works: `->middleware('rate_limit:5,60')` = **`attempts,windowSeconds[,by]`** — the window is in **seconds** there, but in **minutes** in `->rateLimit()`. Don't mix the units up. Use the builder (or `#[RateLimit]`) for tiers, algorithms, or multiple windows. `API_RATE_LIMITING_ENABLED=false` disables all limits.

## Middleware

String middleware names resolve through the **container**. The core aliases are: `admin`, `allow_ip`, `api_version`, `auth`, `auth_to_request`, `conditional_cache`, `csrf`, `enhanced_rate_limit`, `field_selection`, `filter`, `gate_permissions`, `lockdown`, `metrics`, `rate_limit`, `request_logging`, `require_scope`, `security_headers`, `session_cookie`, `tracing`, `validate`. There is **no** `cors`, `role`, `throttle`, or `cache:` alias — don't invent them. A custom middleware must be registered in the container to be referenced by name, and implements `Glueful\Routing\RouteMiddleware`:

```php
public function handle(Request $request, callable $next, mixed ...$params): mixed
```

Keep the `mixed ...$params` variadic — a narrower signature is a fatal interface mismatch. `name:param1,param2` passes the params **as strings** (built-ins parse `"300"`, `"true"`/`"false"`; e.g. `csrf:3600,false`, `require_scope:write:widgets`). Group middleware is inherited by all routes in the group and merges with per-route middleware.

## Conditional registration (feature flags)

Route files run plain PHP, so guard optional routes with an early return. App route files have `$router` and `$context` in scope, so `config($context, 'feature.x')` works; `env()` also works (it casts `'true'`/`'false'` to real booleans):

```php
if (env('FEATURE_X_ENABLED', false) !== true) {
    return;   // routes not registered at all → /feature-x/* returns 404 when off
}

$router->post('/feature-x/run', [FeatureXController::class, 'run'])->middleware(['auth']);
```

## Attribute routing (alternative)

Controllers in `app/Controllers/` are scanned for route attributes at boot (they still need container registration). Attribute routes are **not** API-prefixed — put the full prefix in `#[Controller]`.

```php
use Glueful\Routing\Attributes\{Controller, Get, Post, Middleware, Fields, RequireScope, ResponseStatus};
use Glueful\Api\RateLimiting\Attributes\RateLimit;
use Glueful\Auth\Attributes\RequiresRole;

#[Controller(prefix: '/v1/widgets', middleware: ['auth'])]
final class WidgetController extends BaseController
{
    #[Get('/{id}', where: ['id' => '\d+'])]
    #[Fields(allowed: ['id', 'name', 'owner.name'], strict: true)]
    #[RateLimit(attempts: 60, perMinutes: 1)]
    public function show(int $id): Response { /* ... */ }

    #[Post('/')]
    #[RequireScope('write:widgets')]
    #[RequiresRole('editor')]
    #[ResponseStatus(201)]
    public function store(CreateWidgetData $input): WidgetData { /* ... */ }
}
```

- Verbs: `#[Get]`, `#[Post]`, `#[Put]`, `#[Patch]`, `#[Delete]`, `#[Options]` (`path`, `name`, `where`), or `#[Route(path, methods: [...])]`. `#[Middleware('x')]` adds middleware at class or method level.
- `#[RateLimit(attempts, perMinutes|perHours|perDays, tier, algorithm, by)]` (repeatable, `Glueful\Api\RateLimiting\Attributes`) — auto-attaches `rate_limit`; on the class it applies to every method without its own. `#[RateLimitCost(10)]` weights expensive endpoints.
- `#[Fields(...)]` — field-selection whitelist; auto-attaches `field_selection` (`?fields=`/`?expand=`, strict violations → 400).
- `#[RequireScope('scope')]` — one attribute = any-of its scopes; stacked = all-of; auto-attaches `require_scope`.
- `#[RequiresPermission('x')]` / `#[RequiresRole('x')]` (`Glueful\Auth\Attributes`) — auto-attach `gate_permissions`. Stacked `#[RequiresRole]` = **any one** role; stacked `#[RequiresPermission]` = **all** required.

`#[RateLimit]`/`#[Fields]`/`#[RequireScope]` are only read for **attribute-routed** methods; on a route-file route use `->rateLimit()`, `->middleware('field_selection')`, `->middleware('require_scope:write:widgets')`. (`#[RequiresPermission]`/`#[RequiresRole]` work on both.)

## OpenAPI docs — the reflect generator

The OpenAPI spec is built by the **reflect generator** from the **live route table + PHP types + typed attributes on the controller method** — *not* from route docblocks. The old docblock tags (`@route`, `@summary`, `@requestBody`, `@response`, `@queryParam`) are inert and silently ignored (since 1.57.0), and reflect is the only generator.

```bash
php glueful generate:openapi [--ui[=scalar|swagger-ui|redoc]] [--clean]
php glueful generate:client typescript --spec openapi.json -o ./generated --execute
# language is required (typescript|ts, python, go, ruby, java, …); without --execute it only PRINTS the generator command
```

### What is inferred automatically — no annotation needed

Path, method, and `{param}` path params (with the `->where()` regex as `pattern`); `security` from `auth`/`api_key` middleware (`documentation.middleware_map`) plus auto `401`/`403`; a `429` with rate-limit headers from a rate limit; scope prose from `require_scope`; `fields`/`expand` query params from `#[Fields]`; the **request-body schema** from a typed `RequestData` parameter (plus an auto `422`; its `#[FromRoute]`/`#[FromQuery]` fields become `path`/`query` parameters, and `#[ArrayOf]` drives array `items`); and the **success-response schema** from a `ResponseData` / `CollectionResponse<T>` / `PaginatedResponse<T>` return type. Tags default to the first path segment after the API prefix (`/api/v1/users` → "Users").

> **Type-first rule:** the cleanest way to document an endpoint is to *type* it. A `RequestData` param and a `ResponseData` return need no annotation. See `glueful-build-validation-dto` and `glueful-add-controller`.

### Annotate only what types can't express — on the controller METHOD

The override attributes live in `Glueful\Routing\Attributes\` and go on the **handler method** (not the route file). Each is optional and additive.

```php
use Glueful\Routing\Attributes\{Post, ApiOperation, QueryParam, ApiRequestBody, ApiResponse};

#[Post('/v1/articles/{id}/comments')]
#[ApiOperation(summary: 'Add Comment', description: 'Adds a comment to an article.', tags: ['Comments'])]
#[QueryParam('preview', 'boolean', description: 'Render but do not persist')]
#[ApiRequestBody(schema: CreateCommentData::class)]          // typed DTO class → JSON body schema
#[ApiResponse(201, CommentData::class, description: 'Comment created')]
#[ApiResponse(409, description: 'Duplicate comment')]
public function store(Request $request, int $id): Response { /* ... */ }
```

- **`#[ApiOperation(summary, description, tags, operationId, deprecated)]`** — prose. A non-empty `summary`/`tags` replaces the derived value; `operationId` is generated for you (don't hand-write one).
- **`#[QueryParam(name, type='string', description='', required=false, format=null, enum=null)]`** (repeatable) — `type ∈ string|integer|number|boolean`. Path params auto-derive; `fields`/`expand` come from `#[Fields]` — don't redeclare those.
- **`#[ApiRequestBody(schema|inlineSchema, contentType='application/json', required=true, description='')]`** — exactly one of `schema` (a DTO **class** for a JSON body) or `inlineSchema` (array, only for a non-JSON `contentType` such as `multipart/form-data`).
- **`#[ApiResponse(status, schema=null, description='', collection=false, envelope=true, contentType=..., body=null)]`** (repeatable) — `schema` is a DTO class (`collection: true` wraps it as a list); `body: 'binary'|'text'|'object'` documents a file/HTML/untyped response. Only 401/403 (secured), 422 (`RequestData`), and 429 (rate-limited) are inferred — **declare every other error status here.**

Full reference (in the framework package): `docs/OPENAPI_REFLECT.md`, `docs/REQUEST_DTOS.md`, `docs/RESPONSE_DTOS.md`.

### Gotchas (these have actually bitten)

- **Docs live on the controller method, not the route file.** `@route`/`@response` docblocks above `$router->...` calls are dead (the skeleton's sample route file may still carry some — they do nothing).
- **A typed DTO beats an attribute.** Don't hand-write `#[ApiRequestBody]`/`#[ApiResponse]` for a handler that already takes a `RequestData` param or returns a `ResponseData` — it would only drift.
- **No inline JSON request schema.** `inlineSchema` with `application/json` throws; JSON bodies are a DTO class.
- **Run `php glueful route:cache:clear` on deploy** (and when new routes don't show up). `php glueful route:debug [--method=POST --path=/auth]` lists what's registered.
- **File-level docblock: fully-qualify the `@var`** — `@var \Glueful\Routing\Router $router`.

## Not-Laravel reminders

- No `Route::get(...)` facade → `$router->get(...)` inside a route file.
- No `throttle:60,1` → `->rateLimit(60, 1)` (minutes) or `rate_limit:60,60` (seconds).
- No `Route::resource(...)` / `$router->resource()` → register verbs explicitly, or use `#[Controller]` + verb attributes.
- No route model binding → resolve the entity in the controller from the typed `{id}` argument.
- No `@route`/`@response` docblocks for OpenAPI → typed attributes on the controller method + typed DTOs.

## Checklist

- [ ] Registered on `$router` in a `routes/*.php` file with an explicit prefix group (or `api_prefix($context)`), or via `#[Controller]`/verb attributes with the full prefix.
- [ ] Controller registered in the container (`AppServiceProvider::services()` with `'autowire' => true`).
- [ ] Extension routes load from the provider's `boot()` via `loadRoutesFrom()` (file uses `$router->getContext()` if it needs context).
- [ ] Rate limits via `->rateLimit($attempts, $perMinutes)` / `#[RateLimit]` (or `rate_limit:N,seconds`), units checked.
- [ ] Middleware names are real aliases or container-registered; custom middleware implements `RouteMiddleware::handle(Request, callable, mixed ...$params): mixed`.
- [ ] Path-param constraints via `->where(...)`; route named via `->name(...)`.
- [ ] Optional/feature-flagged routes guarded with an early return.
- [ ] OpenAPI: typed endpoint where possible; `#[ApiOperation]`/`#[QueryParam]`/`#[ApiRequestBody]`/`#[ApiResponse]` on the **controller method** only for prose, query params, manual bodies, uninferred error codes, or non-JSON responses. Ran `generate:openapi` to confirm.
