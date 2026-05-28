---
name: using-glueful
description: This skill should be used when writing or modifying PHP code in a project built on the Glueful framework — controllers, models/ORM queries, routes, middleware, events, services, extensions, or migrations. Glueful is NOT Laravel; this skill provides its real conventions (ApplicationContext, the framework helpers, the ORM and query-builder APIs) and prevents hallucinated Laravel-style APIs. Trigger whenever the codebase namespace is `Glueful\`, depends on `glueful/framework`, or the user mentions Glueful.
---

# Using Glueful

Glueful is a high-performance PHP 8.3+ API framework. It has **its own conventions and APIs** — it is not Laravel, Symfony, or any other framework. The single most common failure mode for an assistant working here is assuming a Laravel-style API exists. Don't.

## Rule 0 — This is NOT Laravel

Do not assume Laravel patterns, syntax, or method names apply. There is **no** `DB::transaction()`, no facade layer, no `Model::create()` static, no `Eloquent`, no `Artisan`, no `Blade`. Before using any method, **verify it exists** in `src/` (or `vendor/glueful/framework/src/`). If you can't find it, it almost certainly doesn't exist — search before writing.

When a convenient method genuinely seems missing: search thoroughly first, then propose adding it to the framework with rationale — never invent or assume one.

## Rule 1 — `ApplicationContext` is required almost everywhere

Most framework entry points take `Glueful\Bootstrap\ApplicationContext` as the **first parameter**. This replaces Laravel-style static facades and global state. If you're calling a helper or a Model static and you don't have a `$context`, that's the thing to resolve first.

## The framework helpers (`src/helpers.php`)

Verified signatures — all take context first:

```php
app(ApplicationContext $context, ?string $abstract = null): mixed      // resolve from container
container(ApplicationContext $context): \Psr\Container\ContainerInterface
db(ApplicationContext $context): \Glueful\Database\Connection
config(ApplicationContext $context, string $key, mixed $default = null): mixed
image(ApplicationContext $context, /* ... */)
```

Usage:

```php
$service = app($context, MyService::class);
$connection = db($context);
$ttl = config($context, 'auth.two_factor.pin_ttl', 300);
```

## Database

Get a query builder via the connection (inject `Connection` or resolve it):

```php
$users = db($context)->table('users')
    ->where('status', '=', 'active')   // 2-arg shorthand only normalizes non-string operands; prefer explicit 3-arg
    ->join('profiles', 'users.id', 'profiles.user_id')
    ->paginate(25);
```

**Transactions** — there is no `DB::transaction()`. Use the connection:

```php
db($context)->transaction(function () use ($context) {
    // ... transactional work
});

db($context)->afterCommit(function () {
    // ... runs only after the surrounding transaction commits
});
```

**Query result caching** (framework ≥ 1.46.0): `->cache(?int $ttl = null, array $tags = [])` caches read queries and tags them by table plus any tags you pass; invalidate with `$cache->invalidateTags([...])`. In earlier versions `->cache()` was a no-op — check the framework version before relying on it.

**Raw query methods** are not all equal for safety:
- `whereRaw`, `havingRaw`, `executeRaw`, `executeRawFirst` accept a bindings array — pass user values through it, never concatenate.
- `selectRaw(string $expression, array $bindings = [])` accepts bindings (framework ≥ 1.45.0).
- `orderByRaw(string $expression)` takes **no** bindings — never pass user input; allowlist it.

## ORM / Models

Model statics take `ApplicationContext` first (this is the part most often gotten wrong):

```php
use Glueful\Bootstrap\ApplicationContext;

$user  = User::find($context, $id);                       // ?static
$query = User::query($context)->where('status', 'active'); // Builder
$users = User::query($context)->with('profile')->paginate(page: 1, perPage: 25);

// Creating / saving
$post = new Post(['title' => 'Hello'], $context);
$post->save();   // bool
```

> Note: some older docs show `User::query()->where(...)` without a context argument. The real signature requires the context — always pass it.

## Controllers

Extend `BaseController` (its constructor takes `ApplicationContext` first). Resolve the authenticated user via the request user context, not Symfony's `$request->getUser()`:

```php
$userUuid = $this->userContext->getUserUuid();   // ?string
$user     = $this->userContext->getUser();        // ?AuthenticatedUser
```

Return the framework response envelope — not raw arrays or `JsonResponse`:

```php
use Glueful\Http\Response;
return Response::success($data, 'Message');   // {success, message, data}
```

## Routing

Routes live in `routes/*.php` (loaded via `RouteManifest`). The builder is fluent; middleware uses the `RouteMiddleware` interface (not PSR-15, though a PSR-15 bridge exists):

```php
$router->post('/users/{id}', [UserController::class, 'show'])
    ->where('id', '\d+')
    ->rateLimit(5, 1)                       // builder form is what's enforced — NOT the rate_limit:N,W string form
    ->middleware(['auth', 'rate_limit'])
    ->name('users.show');
```

Attribute routing exists too (`#[Controller]`, `#[Get]`, `#[Post]`, `#[Fields]`, `#[RequireScope]`).

## Events

All events extend `Glueful\Events\Contracts\BaseEvent` and must call `parent::__construct()`. Dispatch via the injected `EventService` (preferred) or `app($context, EventService::class)`. Register listeners in `config/events.php`.

## Middleware

Implement `Glueful\Routing\Middleware\RouteMiddleware`:

```php
public function handle(Request $request, callable $next, ...$params): Response
```

## Extensions

Extend `Glueful\Extensions\ServiceProvider`. `static services(): array` returns DI definitions; `register()` does runtime wiring (`loadRoutesFrom`, `mergeConfig`); `boot()` runs after all providers (`discoverCommands`, etc.). Scaffold with `php glueful extensions:create <Name>`.

## CLI

Commands extend `Glueful\Console\BaseCommand`, carry a `#[AsCommand]` attribute, and are auto-discovered from `Console/Commands/`. Resolve string-keyed container services (e.g. the connection) with `$this->getServiceDynamic('database')`; class-keyed ones with `$this->getService(Foo::class)`.

## When something seems missing

1. Search `src/` (or `vendor/glueful/framework/src/`) to confirm the method really doesn't exist — check `src/helpers.php` for convenience functions first.
2. If it genuinely doesn't exist and would be valuable, propose adding it to the framework with a clear rationale and implementation sketch.
3. Never fabricate a method name or assume a Laravel equivalent.

## Quick "is this Laravel muscle memory?" checklist

- `DB::`, `Cache::`, `Auth::`, `Route::` static facades → **no**; use `db($context)`, `app($context, …)`, the router instance.
- `Model::create()` / `Model::all()` without context → **no**; `new Model([...], $context)->save()`, `Model::query($context)`.
- `php artisan` → **no**; `php glueful`.
- Blade templates → **no**; this is an API framework (JSON responses via `Response::success`).
- `env()` everywhere in code → prefer `config($context, …)`; `env()` is for config files and bootstrap.
