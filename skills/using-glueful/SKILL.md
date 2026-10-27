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

Most framework entry points take `Glueful\Bootstrap\ApplicationContext` as the **first parameter**. This replaces Laravel-style static facades and global state. If you're calling a helper or a Model static and you don't have a `$context`, that's the thing to resolve first. Where it comes from:

- **Controllers** (`BaseController`), **jobs** (`Queue\Job`): `$this->context`
- **Providers**: `register(ApplicationContext $context)` / `boot(ApplicationContext $context)`
- **Console commands** (`BaseCommand`): `$this->getContext()`
- **App route files** (`routes/*.php`): `$router` and `$context` are injected. Extension route files loaded via `loadRoutesFrom()` get only `$router` — use `$context = $router->getContext();`
- **Middleware / services**: type-hint `ApplicationContext` in the constructor (it's in the container)
- **Config files**: there is no context there — use `env()`

## The framework helpers (`src/helpers.php`)

Verified signatures — context first:

```php
app(ApplicationContext $context, ?string $abstract = null): mixed      // resolve from container
container(ApplicationContext $context): \Psr\Container\ContainerInterface
db(ApplicationContext $context): \Glueful\Database\Connection
config(ApplicationContext $context, string $key, mixed $default = null): mixed   // read-only, not a setter
service(ApplicationContext $context, string $id): mixed
has_service(ApplicationContext $context, string $id): bool
base_path / storage_path / config_path / resource_path(ApplicationContext $context, string $path = ''): string
is_production(ApplicationContext $context): bool
api_prefix(ApplicationContext $context): string     // e.g. "/api/v1"
```

`image()` is **not** a core helper — it's defined by the `glueful/media` extension.

Usage:

```php
$service = app($context, MyService::class);
$connection = db($context);
$ttl = config($context, 'auth.two_factor.pin_ttl', 300);
```

## Database

Get a query builder via the connection (inject `Connection` or resolve it):

```php
$page = db($context)->table('users')
    ->select(['users.uuid', 'users.email'])            // array, not variadic
    ->where('status', '=', 'active')   // 2-arg shorthand only normalizes non-string operands; prefer explicit 3-arg
    ->join('profiles', 'users.uuid', '=', 'profiles.user_uuid')   // join(table, first, operator, second)
    ->paginate(1, 25);                                 // paginate(int $page, int $perPage) — page FIRST
```

- `insert(array)` returns the **row count**; use `insertGetId(array)` for the new id. Bulk: `insertBatch(array $rows)` (no `insertMany`).
- Aggregates: `count()`, `exists()`, `max()`, `pluck()` — there is no `sum()`/`avg()`/`min()`/`find()` on the query builder (use `selectRaw` or the ORM).
- `having(array $conditions)` / `havingRaw(...)` — not `having(col, op, val)`.
- Text search: `whereContains()` / `whereStartsWith()` / `whereEndsWith()` (case-insensitive, escapes `%`/`_`); `whereLike()` takes a raw pattern.

**Transactions** — there is no `DB::transaction()`, and `Connection` has no `beginTransaction()/commit()/rollback()`. Use:

```php
db($context)->transaction(function () use ($context) {
    // ... transactional work; build queries inside the callback (it may be replayed on retry)
});

db($context)->afterCommit(function () {
    // ... runs only after the surrounding transaction commits
});
// also: afterRollback(callable), withinTransaction(): bool
```

**Typed errors**: driver failures surface as `Glueful\Database\Exceptions\*` — e.g. `UniqueConstraintViolationException`, `ForeignKeyConstraintViolationException`, `DeadlockException`, `ConnectionLostException` (unique violations render as 409). Catch these rather than parsing SQLSTATE strings.

**Query result caching**: `->cache(?int $ttl = null, array $tags = [])` caches read queries, tagged by table plus any tags you pass; invalidate with `invalidateTags([...])` on the cache store.

**Raw query methods** are not all equal for safety:
- `whereRaw`, `havingRaw`, `selectRaw`, `executeRaw`, `executeRawFirst` accept a bindings array — pass user values through it, never concatenate.
- `orderByRaw(string $expression)` takes **no** bindings — never pass user input; allowlist it.

## ORM / Models

Model statics take `ApplicationContext` first (this is the part most often gotten wrong):

```php
use Glueful\Bootstrap\ApplicationContext;

$user  = User::find($context, $id);                       // ?static  (also findOrFail, all, firstOrCreate, updateOrCreate, destroy)
$query = User::query($context)->where('status', '=', 'active'); // ORM Builder
$page  = User::query($context)->with('profile')->withCount('posts')->paginate(page: 1, perPage: 25);
// $page['data'] is a Glueful\Database\ORM\Collection of models (eager loads + {relation}_count applied)

// Creating / saving
$post = Post::create($context, ['title' => 'Hello']);
$post = new Post(['title' => 'Hello'], $context);
$post->save();   // bool
```

Soft deletes (`withTrashed()`/`onlyTrashed()`/`restore()`/`forceDelete()`), `belongsToMany` eager loading + `attach/detach/sync/toggle`, `whereHas()` constraint closures, `withCount()`, and model events (returning `false` cancels) all work as named (framework ≥ 1.91).

> Note: a booted app sets a default model context, so a context-less static call may not throw — but the documented signatures are context-first. Always pass `$context`. Factories: `User::factory()->create()` works in a booted app; outside one use `Model::factory($count, $context)` or `Factory::withContext($context)`.

## Controllers

Extend `Glueful\Controllers\BaseController` (constructor: `ApplicationContext $context` first, then optional deps). It exposes `$this->context`, `$this->request`, `$this->currentUser`, `$this->userContext` — there is **no** `$this->db`; use `db($this->context)` or inject `Connection` (better: a repository). Resolve the authenticated user via the request user context, not Symfony's `$request->getUser()`:

```php
$userUuid = $this->userContext->getUserUuid();   // ?string
$user     = $this->userContext->getUser();        // ?Glueful\Auth\UserIdentity ($user->uuid(), $user->email())
```

**Register every controller** in your app provider's `services()` (`Controller::class => ['autowire' => true, 'shared' => true]`) — the router resolves handlers from the container and has no autowire fallback ("Service not found" otherwise). `php glueful scaffold:controller` prints the line to add.

A handler parameter type-hinted with a `Glueful\Validation\Contracts\RequestData` DTO (properties carry `#[Rule('required|string|max:255')]`) is hydrated and validated by the router before the method runs (422 on failure).

> **Identity (framework ≥ 1.50.0):** the concrete user store (`User`/`UserRepository`) was extracted to the `glueful/users` extension; core depends only on the `Glueful\Auth\UserProviderInterface` contract and returns a canonical, immutable `Glueful\Auth\UserIdentity` (accessors are **methods** — `uuid()`, `email()`, `roles()`, `scopes()`). The old `AuthenticatedUser` class no longer exists. With no user store installed, core binds a fail-closed `NullUserProvider` and auth is disabled by design.

Return the framework response envelope — not raw arrays or `JsonResponse`:

```php
use Glueful\Http\Response;
return Response::success($data, 'Message');   // {success, message, data}
```

## Routing

Every `routes/*.php` file in the app is auto-loaded by `RouteManifest` (files starting with `_` are skipped as partials). There is no `Route::` facade and no `$router->resource()`. The builder is fluent; middleware uses the `RouteMiddleware` interface (not PSR-15, though a PSR-15 bridge exists):

```php
$router->group(['prefix' => api_prefix($context), 'middleware' => ['auth']], function (Router $router) {
    $router->get('/users/{id}', [UserController::class, 'show'])
        ->where('id', '\d+')
        ->rateLimit(5, 1)          // (attempts, perMinutes, tier, algorithm, by: 'ip'|'user'|'endpoint')
        ->name('users.show');      // rate_limit middleware is attached automatically
});
```

The string form `rate_limit:attempts,windowSeconds[,ip|user|endpoint]` also works (note: window in **seconds**, vs minutes in `->rateLimit()`). Real middleware aliases: `auth`, `rate_limit`, `enhanced_rate_limit`, `csrf`, `security_headers`, `field_selection`, `require_scope`, `gate_permissions`, `admin`, `allow_ip`, `lockdown`, `conditional_cache`, `request_logging`, `session_cookie`, `metrics`, `tracing`, `validate`, `auth_to_request`, `filter`. There is **no** `cors`, `role`, or `cache:` alias (CORS is applied globally from `config/cors.php`).

Attribute routing exists too — `Glueful\Routing\Attributes\{Controller, Get, Post, Put, Patch, Delete, Middleware, Fields, RequireScope}`, plus `#[RateLimit]` (`Glueful\Api\RateLimiting\Attributes`) and `#[RequiresPermission]` / `#[RequiresRole]` (`Glueful\Auth\Attributes`; stacked roles = any-of). Controllers in `app/Controllers/` are scanned at boot; attribute routes are **not** API-prefixed, so put the full prefix in `#[Controller]`. `#[Fields]`, `#[RateLimit]` and the gate attributes auto-attach their middleware.

## Events

Events extend `Glueful\Events\Contracts\BaseEvent` and call `parent::__construct()`. Dispatch via the injected `Glueful\Events\EventService` (`dispatch($event)`) or `app($context, EventService::class)`. Listeners are **not** registered in `config/events.php` (that file only toggles core listeners). Mark handler methods `#[AsListener(SomeEvent::class)]` (`Glueful\Events\Attributes\AsListener`, or implement `EventSubscriberInterface`) and subscribe the class from a provider's `boot()`: `app($context, EventService::class)->subscribe(MyListener::class)`. Ad-hoc: `addListener(SomeEvent::class, $callable)`. Mark an event `#[Webhookable]` to fan it out to webhook subscribers — no listener needed.

## Middleware

Implement `Glueful\Routing\RouteMiddleware` (Symfony `Request`); keep the variadic — dropping it is a fatal:

```php
public function handle(Request $request, callable $next, mixed ...$params): mixed
```

String params (`my_mw:a,b`) arrive as **strings**. Register the class in `services()` and reference it by FQCN (or a container id you define).

## Official extensions — prefer these over rolling your own

Before hand-building a capability, check whether an **official Glueful extension** already covers it (catalog: <https://glueful.com/extensions>). Install via Composer and enable; don't reimplement these in app code:

| Extension | Package | Use for |
| --------- | ------- | ------- |
| Users | `glueful/users` | **First-party identity store & account lifecycle** — the concrete `UserProviderInterface` implementation (`users`/`profiles` tables, credential verification, email verification/OTP, password reset, optional email-PIN 2FA, `/me` + `/users` endpoints). Core ships **no** user store; install this (or another `UserProviderInterface`) to enable authentication. The api-skeleton enables it by default. |
| Aegis | `glueful/aegis` | Role-based access control — roles, permissions, authorization workflows |
| Tenancy | `glueful/tenancy` | Shared-database, row-level multi-tenancy — tenant-owned tables (`tenant_uuid` + `BelongsToTenant`), automatic per-tenant scoping, memberships, resolution middleware, and explicit bypass APIs |
| Entrada | `glueful/entrada` | Social login / SSO — OAuth & OpenID Connect flows |
| Audit | `glueful/audit` | Event-sourced audit trail (audit logging is not in core) |
| Email Notification | `glueful/email-notification` | Email delivery (Symfony Mailer) — transactional notifications; the email channel for core features like 2FA |
| Notiva | `glueful/notiva` | Push notifications — FCM, APNs, Web Push |
| Meilisearch | `glueful/meilisearch` | Full-text search / external indexing |
| Payvia | `glueful/payvia` | Payments — Stripe, Paystack, Flutterwave, and other gateways |
| Runiva | `glueful/runiva` | Server runtime — RoadRunner, Swoole, FrankenPHP |
| Media | `glueful/media` | Image processing / thumbnails / media metadata (Intervention Image + getID3) — **binds the core `MediaProcessorInterface` seam**. Without it `uploadMedia()` yields no thumbnail + type-only metadata and the `image()` helper is undefined. |
| CDN / Edge Cache | `glueful/cdn` | Edge cache-control headers + content purge via provider adapters — **binds the core `EdgeCacheInterface` seam** (core defaults to a no-op `NullEdgeCache`). |
| Queue Ops | `glueful/queue-ops` | Supervised worker fleets, autoscaling, worker/job metrics (`queue:supervise`, `queue:autoscale`) — **binds the core `WorkerMonitorInterface` seam**. Core ships only a lean single-worker `queue:work`. |
| Archive | `glueful/archive` | Generic table archiving — archive / restore / search with a registry + schema (`archive:manage`). Opt-in; ships migrations. |
| I18n | `glueful/i18n` | Platform localization — locale registry with single-parent fallback chains, request locale resolver, persisted translation catalogs, pluralization, missing-key tracking, management API/CLI. **Binds the `translation.manager` seam**: every extension's `loadMessageCatalogs()` is a silent no-op until this is installed. |
| Import/Export | `glueful/import-export` | Bulk data engine — jobs, deterministic queue-backed batches, dry-run/commit modes, row-error reports, retries, streaming CSV/JSON/NDJSON/ZIP-bundle readers and writers, HTTP/CLI job management. Your app implements `ImporterInterface`/`ExporterInterface` adapters (registered by service tag) for its record types; the engine knows nothing about your domain. |
| Flags | `glueful/flags` | Feature flags — definitions, targeting rules, deterministic percentage rollouts, management API/CLI, audit rows, lifecycle events. A rollout switchboard (dark launches, kill switches) — explicitly *not* access control or billing. |
| Storage packs | `glueful/storage-s3` · `storage-gcs` · `storage-azure` | Object-store disks — each registers a `StorageDriverFactoryInterface` on the core storage registry seam (`storage.driver_factory` tag). S3 pack also covers R2, MinIO, Spaces, Wasabi as S3-compatible targets. Disks with `driver => s3|gcs|azure` resolve once installed. |
| Conversa | `glueful/conversa` | SMS & WhatsApp notification channels — swappable provider drivers (Twilio, Meta WhatsApp Cloud, log), message log, delivery tracking. |
| Subscriptions | `glueful/subscriptions` | Plans, tenant subscriptions, and stateless entitlements — numeric limits, trials, overrides, lifecycle sync for SaaS apps. |
| Commerce | `glueful/commerce` | Commerce primitives — products, carts, orders, inventory, discounts, checkout, payment integration (pairs with Payvia). Ships no UI. |

Media, CDN, Queue Ops, and Archive were **extracted from core in framework 1.52.0**: core keeps a contract + no-op default (a *seam*) and the extension binds the real implementation when installed (see `glueful-create-extension` → "Activating an optional core capability"). If the user asks for RBAC, social login, audit trails, email, push, SMS/WhatsApp, search, payments, a high-performance runtime, image processing, a CDN/edge cache, queue supervision/autoscaling, table archiving, localization, feature flags, bulk import/export, S3/GCS/Azure storage, or subscription plans/entitlements, recommend the matching extension rather than scaffolding bespoke code. Build a custom extension only when no official one fits.

## Extensions

Extend `Glueful\Extensions\ServiceProvider`. `static services(): array` returns DI definitions; `register(ApplicationContext $context)` is for lightweight registration only, especially config defaults (`mergeConfig($key, $defaults)` — not `mergeConfigFrom`). Runtime wiring belongs in `boot(ApplicationContext $context)`, after all providers are registered: routes (`loadRoutesFrom`), commands (`discoverCommands($namespace, $dir)` / `commands([...])`), event subscriptions, message catalogs, notification channels, a built SPA (`serveFrontend('/admin', $dir)`). Providers can also declare `permissions()`, `roles()`, `voters()`, `policies()`.

**Migrations are declared, not just loaded**: a package declares them in composer `extra.glueful.migrations` (rows with `id`, `path` — a flat `migrations/` folder at the package root — `priority`, `mode: core|on_enable`), or `"migrations": "none"`. An installed package without that declaration can't register migrations (`loadMigrationsFrom()` throws `UndeclaredSchemaException`); only app-local providers keep the plain `loadMigrationsFrom()` lane. `extensions:enable` runs the package's migrations first, then enables it.

Discovery is via the package's composer `extra.glueful.provider` (package `type: glueful-extension`). Scaffold with `php glueful create:extension <name>` (note: `create:extension`, **not** `extensions:create`); manage installed ones with `extensions:enable|disable|list|info|diagnose|summary|cache`.

## CLI

The console is `php glueful <command>` (artisan's analogue — but **not** `php artisan`). The framework also ships an executable bin that Composer symlinks, so `vendor/bin/glueful <command>` runs it **without `php`**. Prefer `php glueful` in docs/examples unless the project standardizes on the bin; the command set comes from the app + its enabled extensions.

Common: `migrate:create|run|rollback|status`, `db:seed` (runs `App\Database\Seeders\DatabaseSeeder` by default), `scaffold:controller|model|dto|request|resource|middleware|job|rule|filter|factory|seeder|test`, `event:create|listener`, `queue:work|failed|retry|restart`, `queue:scheduler` (jobs live in `config/schedule.php`), `route:debug`, `route:cache:clear`, `generate:openapi`, `doctor`. There is no `make:*`, `config:cache` or `config:clear`.

Writing commands: extend `Glueful\Console\BaseCommand` with an `#[AsCommand]` attribute. App commands are **not** auto-discovered — register them from a provider's `boot()` with `$this->discoverCommands('App\\Console\\Commands', base_path($context, 'app/Console/Commands'))` or `$this->commands([...])`. In a command, use `$this->getContext()`; resolve string-keyed services with `$this->getServiceDynamic('database')`, class-keyed ones with `$this->getService(Foo::class)`.

## When something seems missing

1. Search `src/` (or `vendor/glueful/framework/src/`) to confirm the method really doesn't exist — check `src/helpers.php` for convenience functions first.
2. If it genuinely doesn't exist and would be valuable, propose adding it to the framework with a clear rationale and implementation sketch.
3. Never fabricate a method name or assume a Laravel equivalent.

## Quick "is this Laravel muscle memory?" checklist

- `DB::`, `Cache::`, `Auth::`, `Route::` static facades → **no**; use `db($context)`, `app($context, …)`, the router instance.
- `Model::create([...])` / `Model::all()` / `Model::query()` *without context* → **no**; they exist but are context-first: `Model::create($context, [...])`, `Model::query($context)`, or `new Model([...], $context)->save()`.
- `$this->db` in a controller, `$connection->beginTransaction()` → **no**; `db($this->context)`, `->transaction(fn)`.
- `$router->resource()`, `Storage::put`, `view()`, `now()`, `logger()`, `auth()->user()` → **no**; explicit routes, injected services, `$this->userContext`.
- `php artisan` / `make:*` → **no**; `php glueful` / `scaffold:*`.
- Blade templates → **no**; this is an API framework (JSON responses via `Response::success`).
- `env()` everywhere in code → prefer `config($context, …)`; `env()` is for config files and bootstrap. Env names: `BASE_URL` (not `APP_URL`), `JWT_KEY` (not `JWT_SECRET`), `DB_DRIVER` (not `DB_CONNECTION`).
