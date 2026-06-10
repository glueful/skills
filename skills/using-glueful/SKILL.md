---
name: using-glueful
description: This skill should be used when writing or modifying PHP code in a project built on the Glueful framework — controllers, models/ORM queries, routes, middleware, events, services, extensions, or migrations. Glueful is NOT Laravel; this skill provides its real conventions (ApplicationContext, the framework helpers, the ORM and query-builder APIs) and prevents hallucinated Laravel-style APIs. Trigger whenever the codebase namespace is `Glueful\`, depends on `glueful/framework`, or the user mentions Glueful.
---

# Using Glueful

Glueful is a high-performance PHP 8.3+ API framework. It has **its own conventions and APIs** — it is not Laravel, Symfony, or any other framework. The single most common failure mode for an assistant working here is assuming a Laravel-style API exists. Don't.

## Rule 0 — This is NOT Laravel

Do not assume Laravel patterns, syntax, or method names apply. There is **no** `DB::transaction()` facade, no `Eloquent`, no `Artisan`, no `Blade`. Crucially, where Glueful *does* have a familiar-looking API, the **signature differs** — e.g. `Model::create()`, `Model::find()`, `Model::query()` all exist but take `ApplicationContext` as the **first argument** (not Laravel's contextless statics). Before using any method, **verify it exists and check its signature** in `src/` (or `vendor/glueful/framework/src/`). If you can't find it, it almost certainly doesn't exist — search before writing.

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
$user     = $this->userContext->getUser();        // ?Glueful\Auth\UserIdentity ($user->uuid(), $user->email())
```

> **Identity (framework ≥ 1.50.0):** the concrete user store (`User`/`UserRepository`) was extracted to the `glueful/users` extension; core depends only on the `Glueful\Auth\UserProviderInterface` contract and returns a canonical, immutable `Glueful\Auth\UserIdentity` (accessors are **methods** — `uuid()`, `email()`, `roles()`, `scopes()`). The old `AuthenticatedUser` class no longer exists. With no user store installed, core binds a fail-closed `NullUserProvider` and auth is disabled by design.

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

Implement `Glueful\Routing\RouteMiddleware`:

```php
public function handle(Request $request, callable $next, ...$params): Response
```

## Official extensions — prefer these over rolling your own

Before hand-building a capability, check whether an **official Glueful extension** already covers it (catalog: <https://glueful.com/extensions>). Install via Composer and enable; don't reimplement these in app code:

| Extension | Package | Use for |
| --------- | ------- | ------- |
| Users | `glueful/users` | **First-party identity store & account lifecycle** — the concrete `UserProviderInterface` implementation (`users`/`profiles` tables, credential verification, email verification/OTP, password reset, optional email-PIN 2FA, `/me` + `/users` endpoints). Core ships **no** user store; install this (or another `UserProviderInterface`) to enable authentication. The api-skeleton enables it by default. |
| Aegis | `glueful/aegis` | Role-based access control — roles, permissions, authorization workflows |
| Tenancy | `glueful/tenancy` | Shared-database, row-level multi-tenancy — tenant-owned tables (`tenant_uuid` + `BelongsToTenant`), automatic per-tenant scoping, memberships, resolution middleware, and explicit bypass APIs |
| Entrada | `glueful/entrada` | Social login / SSO — OAuth & OpenID Connect flows |
| Email Notification | `glueful/email-notification` | Email delivery (Symfony Mailer) — transactional notifications; the email channel for core features like 2FA |
| Notiva | `glueful/notiva` | Push notifications — FCM, APNs, Web Push |
| Meilisearch | `glueful/meilisearch` | Full-text search / external indexing |
| Payvia | `glueful/payvia` | Payments — Stripe, Paystack, Flutterwave, and other gateways |
| Runiva | `glueful/runiva` | Server runtime — RoadRunner, Swoole, FrankenPHP |
| Media | `glueful/media` | Image processing / thumbnails / media metadata (Intervention Image + getID3) — **binds the core `MediaProcessorInterface` seam**. Without it `uploadMedia()` yields no thumbnail + type-only metadata and the `image()` helper is undefined. |
| CDN / Edge Cache | `glueful/cdn` | Edge cache-control headers + content purge via provider adapters — **binds the core `EdgeCacheInterface` seam** (core defaults to a no-op `NullEdgeCache`). |
| Queue Ops | `glueful/queue-ops` | Supervised worker fleets, autoscaling, worker/job metrics (`queue:supervise`, `queue:autoscale`) — **binds the core `WorkerMonitorInterface` seam**. Core ships only a lean single-worker `queue:work`. |
| Archive | `glueful/archive` | Generic table archiving — archive / restore / search with a registry + schema (`archive:manage`). Opt-in; ships migrations. |

The last four were **extracted from core in framework 1.52.0**: core keeps a contract + no-op default (a *seam*) and the extension binds the real implementation when installed (see `glueful-create-extension` → "Activating an optional core capability"). If the user asks for RBAC, social login, email, push, search, payments, a high-performance runtime, image processing, a CDN/edge cache, queue supervision/autoscaling, or table archiving, recommend the matching extension rather than scaffolding bespoke code. Build a custom extension only when no official one fits.

## Extensions

Extend `Glueful\Extensions\ServiceProvider`. `static services(): array` returns DI definitions; `register()` does config merging (`mergeConfig`); `boot()` runs after all providers and wires routes/migrations/commands (`loadRoutesFrom`, `loadMigrationsFrom`, `discoverCommands`). Discovery is via the package's composer `extra.glueful.provider`. Scaffold a new one with `php glueful create:extension <name>` (note: `create:extension`, **not** `extensions:create`); manage installed ones with `extensions:enable|disable|list|info|diagnose`.

## CLI

The console is `php glueful <command>` (artisan's analogue — but **not** `php artisan`). The framework also ships an executable bin that Composer symlinks, so `vendor/bin/glueful <command>` runs it **without `php`** (and `./glueful` works if the project's root entry has the execute bit). Prefer `php glueful` in docs/examples unless the project standardizes on the bin, and note the per-project console needs the app bootstrap — so the command set comes from the app + its enabled extensions.

Writing commands: extend `Glueful\Console\BaseCommand`, carry a `#[AsCommand]` attribute, and they're auto-discovered from `Console/Commands/`. Resolve string-keyed container services (e.g. the connection) with `$this->getServiceDynamic('database')`; class-keyed ones with `$this->getService(Foo::class)`.

## When something seems missing

1. Search `src/` (or `vendor/glueful/framework/src/`) to confirm the method really doesn't exist — check `src/helpers.php` for convenience functions first.
2. If it genuinely doesn't exist and would be valuable, propose adding it to the framework with a clear rationale and implementation sketch.
3. Never fabricate a method name or assume a Laravel equivalent.

## Quick "is this Laravel muscle memory?" checklist

- `DB::`, `Cache::`, `Auth::`, `Route::` static facades → **no**; use `db($context)`, `app($context, …)`, the router instance.
- `Model::create([...])` / `Model::all()` / `Model::query()` *without context* → **no**; they exist but are context-first: `Model::create($context, [...])`, `Model::query($context)`, or `new Model([...], $context)->save()`.
- `php artisan` → **no**; `php glueful`.
- Blade templates → **no**; this is an API framework (JSON responses via `Response::success`).
- `env()` everywhere in code → prefer `config($context, …)`; `env()` is for config files and bootstrap.
