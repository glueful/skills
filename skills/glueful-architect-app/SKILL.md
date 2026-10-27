---
name: glueful-architect-app
description: Decide where code belongs and how it wires together when building a non-trivial feature in a Glueful application — the layered structure (controllers → DTOs → services → repositories/contracts → models), domain events/listeners/jobs, query filters, app DI via AppServiceProvider, and reusing official extensions. Use when adding a feature, placing new code, wiring providers, or reviewing app structure. This is the map of HOW a serious Glueful app is composed; the per-task skills are the implementation guides.
---

# Architecting a Glueful App

This is the **placement and wiring map** for a non-trivial Glueful app — where each kind of code goes and how the layers connect. It's a *recommended* production structure built entirely on framework primitives (not framework-mandated, but how serious Glueful apps are composed). For the code recipes of any one layer, use the focused skill (controllers, routes, ORM, migrations, tests, filters, events/jobs, DTOs).

> **Reflect the current framework.** Where the framework's current capability differs from an older app's habits, follow the framework — e.g. listeners as `#[AsListener]` classes subscribed via `EventService::subscribe()` (`config/events.php` has no listener map), request DTOs implementing `RequestData` that the router hydrates and validates, and the `Api/Filtering` subsystem for list filtering. Prefer the newest supported mechanism.

## Step 0 — reuse official extensions before building

Before adding a layer, check whether an official extension already owns the capability (identity/accounts → `glueful/users`, RBAC → `glueful/aegis`, row-level multi-tenancy → `glueful/tenancy`, social/SSO → `glueful/entrada`, audit trail → `glueful/audit`, email → `glueful/email-notification`, push → `glueful/notiva`, SMS/WhatsApp → `glueful/conversa`, search → `glueful/meilisearch`, payments → `glueful/payvia`, commerce → `glueful/commerce`, plans/entitlements → `glueful/subscriptions`, runtime → `glueful/runiva`, image processing → `glueful/media`, edge cache → `glueful/cdn`, queue supervision/autoscaling → `glueful/queue-ops`, table archiving → `glueful/archive`, localization → `glueful/i18n`, feature flags → `glueful/flags`, bulk import/export → `glueful/import-export`, S3/GCS/Azure disks → `glueful/storage-s3|gcs|azure`). Don't reimplement framework- or extension-provided endpoints (auth, RBAC, health, RESTful CRUD). See `using-glueful` → "Official extensions."

Core also already ships: outbound **webhooks** (mark an event `#[Webhookable]`; subscriptions/deliveries/`/webhooks` management routes are built in), file uploads/blobs, API keys, notifications, the scheduler (`config/schedule.php`), API resources (`Http\Resources`), and OpenAPI generation from route docblocks. Use them rather than building parallel versions.

## The layers

| Layer | Responsibility | Backed by | Keep OUT |
| ----- | -------------- | --------- | -------- |
| **Controller** | HTTP only: read request, call a service, shape the response | `BaseController`, `Response`, `Http\Resources\JsonResource` | business logic, SQL, validation rules |
| **DTO** | Validate + normalize request input | `Validation\Contracts\RequestData` + `#[Rule]` (router-hydrated); or `Validator` + `ValidationException` in a factory | persistence, orchestration |
| **Service** | Business orchestration; the unit of "a use case"; dispatches domain events | plain DI-wired classes | HTTP concerns, direct SQL/query-builder |
| **Repository** (+ Contract) | All data access (query builder / models) behind an interface | `BaseRepository`, `RepositoryInterface`, `QueryFilterTrait` | business rules, HTTP |
| **Model** | Entity + relations | ORM `Model` | service logic |
| **Filter** | Filtering/sorting/search for list endpoints | `Api/Filtering` (`QueryFilter`, `FilterableInterface`) / `QueryFilterTrait` | business rules |
| **Event** | A domain fact that happened (`UserRegistered`) | `Events\Contracts\BaseEvent` | side-effect code |
| **Listener / Subscriber** | Synchronous side effects of an event | `#[AsListener]` methods or `EventSubscriberInterface` | heavy/slow work (push to a job) |
| **Job** | Async / deferred work (notifications, counters, re-indexing) | `Queue\Job` (`$this->context`, `getData()`), `QueueManager::push()` | request-path logic |
| **Provider** | DI registration (plain `services()` class) and/or lifecycle wiring (extends `ServiceProvider`) | `App\Providers\*`; lifecycle ones extend `Extensions\ServiceProvider` | feature logic |

## Request flow

```
HTTP request
  → Router hydrates + validates a RequestData DTO param  (422 before the controller runs)
  → Controller          (BaseController; resolve user via $this->userContext)
      → Service          (orchestrate the use case)
          → Repository    (data access behind a Contract; QueryFilter for lists)
              → Model / query builder
          → dispatch domain Event(s)   (BaseEvent, from the service — not the controller)
      ← Service returns data/result
  ← Controller returns Response::success(...) / Resource::make(...)->toResponse()
```

- **Write path:** controller receives a validated DTO → hands it to a service → service uses repositories + dispatches events.
- **Read/list path:** controller → service → repository, applying a **Filter** for `?filter=`/sort/search.
- **Side effects** ride on events: a **listener** reacts synchronously; anything heavy (email, push, search re-index) is queued as a **Job**. Outbound webhooks need no listener — mark the event `#[Webhookable]`.

## Dependency injection (app wiring)

There are **two kinds of app provider**, and the distinction matters:

- **DI provider** (e.g. `AppServiceProvider`) — a **plain class** (no base class) exposing a static `services(): array`. The container's services loader discovers it and registers the definitions. This is *only* DI; it does **not** need to extend anything.
- **Lifecycle provider** (e.g. `EventServiceProvider`) — **extends `Glueful\Extensions\ServiceProvider`** so it gets `register()`/`boot()` for runtime wiring like `EventService::subscribe()`. The runtime provider/extension machinery requires the `ServiceProvider` base for these.

Both are listed in `config/serviceproviders.php` (the api-skeleton ships `AppServiceProvider` enabled and an `EventServiceProvider` line commented out). **Every controller must be registered** here — the router resolves handlers from the container with no autowire fallback. In the DI provider, bind a repository **contract** to its implementation with the `alias` key on the *concrete* service so services can type-hint the interface (the reverse direction, `Interface => ['alias' => Concrete]`, does not work):

```php
// app/Providers/AppServiceProvider.php  — plain DI provider, NO base class
final class AppServiceProvider
{
    public static function services(): array
    {
        return [
            \App\Controllers\ArticleController::class => ['autowire' => true, 'shared' => true],

            \App\Services\Article\ArticleService::class => ['autowire' => true, 'shared' => true],

            \App\Repositories\Article\ArticleRepository::class => [
                'autowire' => true,
                'shared'   => true,
                'alias'    => [\App\Repositories\Contracts\ArticleRepositoryInterface::class], // contract → impl
            ],
        ];
    }
}
```

```php
// config/serviceproviders.php  →  single `enabled` list of app providers (order preserved)
return [
    'enabled' => [
        // plain string FQCNs (no ::class) so tooling can edit the list safely
        'App\\Providers\\AppServiceProvider',
        'App\\Providers\\EventServiceProvider',
    ],
];
```

A **lifecycle** provider extends `ServiceProvider` and wires listeners (and app console commands) in `boot()`:

```php
// app/Providers/EventServiceProvider.php  — lifecycle provider, EXTENDS ServiceProvider
use App\Events\Listeners\ArticleSearchListener;
use Glueful\Bootstrap\ApplicationContext;
use Glueful\Events\EventService;
use Glueful\Extensions\ServiceProvider;

final class EventServiceProvider extends ServiceProvider
{
    public function boot(ApplicationContext $context): void
    {
        // registers the class's #[AsListener] methods (and/or EventSubscriberInterface)
        app($context, EventService::class)->subscribe(ArticleSearchListener::class);

        // app commands are not auto-discovered
        $this->discoverCommands('App\\Console\\Commands', base_path($context, 'app/Console/Commands'));
    }
}
```

```php
// app/Events/Listeners/ArticleSearchListener.php
final class ArticleSearchListener
{
    #[\Glueful\Events\Attributes\AsListener(ArticleCreated::class)]
    public function onCreated(ArticleCreated $event): void { /* queue a re-index Job */ }
}
```

Don't subscribe the framework's `WebhookEventListener` yourself — core registers it at boot, and doing it again sends each webhook twice.

> Both providers are listed in `config/serviceproviders.php` (app-level, loaded in declared order). **Packaged extensions** instead declare their provider via composer `extra.glueful.provider` (plus an `extra.glueful.migrations` declaration — see `glueful-create-extension`) — and an extension provider always extends `ServiceProvider`.

## Recommended `app/` layout

```
app/
├── Controllers/         # HTTP entry points (scanned for attribute routes) (+ Traits/)
├── DTOs/                # RequestData / ResponseData DTOs (group by domain)
├── Http/
│   ├── Middleware/      # RouteMiddleware classes
│   └── Resources/       # JsonResource / ModelResource transformers
├── Services/            # business orchestration (group by domain)
├── Repositories/
│   └── Contracts/       # repository interfaces (bound via 'alias')
├── Models/              # ORM models
├── Filters/             # QueryFilter classes for list endpoints
├── Events/              # domain events (BaseEvent)
│   └── Listeners/       # #[AsListener] classes (sync side effects)
├── Jobs/                # queued async work
├── Validation/Rules/    # custom validation rules
├── Console/Commands/    # app commands (registered via discoverCommands)
├── Database/            # Factories/, Seeders/ (App\Database\…)
└── Providers/           # AppServiceProvider, EventServiceProvider
```

These match where the `scaffold:*` / `event:*` generators write (`scaffold:dto` → `app/DTOs`, `event:listener` → `app/Events/Listeners`, etc.); Services/Repositories have no scaffold. Migrations live in `database/migrations/` at the project root; routes in `routes/*.php` (all auto-loaded).

## Boundaries (the "where does it go?" rules)

- **Controllers stay thin** — no business logic, no SQL. If a controller method is doing real work, that work belongs in a service.
- **Validation lives in DTOs**, not in controllers (prevents per-endpoint validation drift). `BaseController::validateRequest()` is legacy (only `required`/`email`/`max:N`) — don't build on it.
- **Only repositories touch the database.** Services and controllers never build queries directly; they call repositories typed against contracts.
- **Services own use-cases and emit events**; they don't handle HTTP or know about requests/responses.
- **Side effects are events, not inline calls.** A service dispatches `ArticleCreated`; listeners send notifications / bump counters and defer heavy work to jobs; `#[Webhookable]` on the event handles outbound webhooks. Dispatch inside a transaction only what's safe to roll back — use `db($context)->afterCommit(...)` for anything that must not fire on rollback.
- **List endpoints use a Filter**, not hand-rolled `where()` chains in the controller.
- **Reach for an official extension** before building cross-cutting infrastructure (auth, RBAC, search, payments…).

## Implementation guides (per layer)

- Controllers → `glueful-add-controller` · Routes/docs → `glueful-add-route` · Data/queries → `glueful-build-orm-query` · Schema → `glueful-write-migration` · Tests → `glueful-write-test`
- Input validation → `glueful-build-validation-dto` · List filtering → `glueful-add-filter` · Side effects/async → `glueful-add-event-listener-job` · Packaging → `glueful-create-extension`

## Checklist (placing a new feature)

- [ ] Checked official extensions first; not duplicating package-provided functionality.
- [ ] HTTP only in the controller; input validated in a DTO (`RequestData` param).
- [ ] Business logic in a service; all data access in a repository behind a contract.
- [ ] Controller registered and repository bound to its interface via `'alias'` in `AppServiceProvider::services()`; providers listed in `config/serviceproviders.php`.
- [ ] Side effects dispatched as domain events; `#[AsListener]` classes subscribed via `EventService::subscribe()` in a provider's `boot()`; heavy work in jobs; webhooks via `#[Webhookable]`.
- [ ] List endpoints use a Filter for filtering/sorting/search.
