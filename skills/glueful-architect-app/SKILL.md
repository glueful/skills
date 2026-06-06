---
name: glueful-architect-app
description: Decide where code belongs and how it wires together when building a non-trivial feature in a Glueful application — the layered structure (controllers → DTOs → services → repositories/contracts → models), domain events/listeners/jobs, query filters, app DI via AppServiceProvider, and reusing official extensions. Use when adding a feature, placing new code, wiring providers, or reviewing app structure. This is the map of HOW a serious Glueful app is composed; the per-task skills are the implementation guides.
---

# Architecting a Glueful App

This is the **placement and wiring map** for a non-trivial Glueful app — where each kind of code goes and how the layers connect. It's a *recommended* production structure built entirely on framework primitives (not framework-mandated, but how serious Glueful apps are composed). For the code recipes of any one layer, use the focused skill (controllers, routes, ORM, migrations, tests, filters, events/jobs, DTOs).

> **Reflect the current framework.** Where the framework's current capability differs from an older app's habits, follow the framework — e.g. event **subscribers** via `EventService::subscribe()` (not just `config/events.php` listener arrays), and the `Api/Filtering` subsystem for list filtering. Prefer the newest supported mechanism.

## Step 0 — reuse official extensions before building

Before adding a layer, check whether an official extension already owns the capability (identity/accounts → `glueful/users`, RBAC → `glueful/aegis`, social/SSO → `glueful/entrada`, email → `glueful/email-notification`, push → `glueful/notiva`, search → `glueful/meilisearch`, payments → `glueful/payvia`, runtime → `glueful/runiva`). Don't reimplement framework- or extension-provided endpoints (auth, RBAC, health, RESTful CRUD). See `using-glueful` → "Official extensions."

## The layers

| Layer | Responsibility | Backed by | Keep OUT |
| ----- | -------------- | --------- | -------- |
| **Controller** | HTTP only: read request, call a service, shape the response | `BaseController`, `Response` | business logic, SQL, validation rules |
| **DTO** | Validate + normalize request input | `Validator`, `ValidationException`, `Contracts\Rule` | persistence, orchestration |
| **Service** | Business orchestration; the unit of "a use case"; dispatches domain events | plain DI-wired classes | HTTP concerns, direct SQL/query-builder |
| **Repository** (+ Contract) | All data access (query builder / models) behind an interface | `BaseRepository`, `RepositoryInterface`, `QueryFilterTrait` | business rules, HTTP |
| **Model** | Entity + relations | ORM `Model` | service logic |
| **Filter** | Filtering/sorting/search for list endpoints | `Api/Filtering` (`QueryFilter`, `FilterableInterface`) / `QueryFilterTrait` | business rules |
| **Event** | A domain fact that happened (`UserRegistered`) | `Events\Contracts\BaseEvent` | side-effect code |
| **Listener / Subscriber** | Synchronous side effects of an event | `EventSubscriberInterface` | heavy/slow work (push to a job) |
| **Job** | Async / deferred work (notifications, counters, webhooks) | `Queue\Job`, `QueueManager` | request-path logic |
| **Provider** | DI registration (plain `services()` class) and/or lifecycle wiring (extends `ServiceProvider`) | `App\Providers\*`; lifecycle ones extend `Extensions\ServiceProvider` | feature logic |

## Request flow

```
HTTP request
  → Controller          (BaseController; resolve user via $this->userContext)
      → DTO::from($data) (validate + normalize; throws ValidationException)
      → Service          (orchestrate the use case)
          → Repository    (data access behind a Contract; QueryFilter for lists)
              → Model / query builder
          → dispatch domain Event(s)   (BaseEvent, from the service — not the controller)
      ← Service returns data/result
  ← Controller returns Response::success(...) / a Resource
```

- **Write path:** controller validates via DTO → hands to a service → service uses repositories + dispatches events.
- **Read/list path:** controller → service → repository, applying a **Filter** for `?filter=`/sort/search.
- **Side effects** ride on events: a **subscriber/listener** reacts synchronously; anything heavy (email, push, search re-index, webhooks) is queued as a **Job**.

## Dependency injection (app wiring)

There are **two kinds of app provider**, and the distinction matters:

- **DI provider** (e.g. `AppServiceProvider`) — a **plain class** (no base class) exposing a static `services(): array`. The container's services loader discovers it and registers the definitions. This is *only* DI; it does **not** need to extend anything.
- **Lifecycle provider** (e.g. `EventServiceProvider`) — **extends `Glueful\Extensions\ServiceProvider`** so it gets `register()`/`boot()` for runtime wiring like `EventService::subscribe()`. The runtime provider/extension machinery requires the `ServiceProvider` base for these.

Both are listed in `config/serviceproviders.php`. In the DI provider, bind a repository **contract** to its implementation with the `alias` key so services can type-hint the interface:

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

A **lifecycle** provider extends `ServiceProvider` and wires subscribers in `register()`/`boot()` via `EventService::subscribe()`:

```php
// app/Providers/EventServiceProvider.php  — lifecycle provider, EXTENDS ServiceProvider
use Glueful\Bootstrap\ApplicationContext;
use Glueful\Extensions\ServiceProvider;

final class EventServiceProvider extends ServiceProvider
{
    public function boot(ApplicationContext $context): void
    {
        $events = app($context, \Glueful\Events\EventService::class);
        $events->subscribe(\App\Listeners\SearchCounterListener::class);
        $events->subscribe(\App\Listeners\WebhookEventListener::class);
    }
}
```

> Both providers are listed in `config/serviceproviders.php` (app-level discovery). **Packaged extensions** instead declare their provider via composer `extra.glueful.provider` (see `glueful-create-extension`) — and an extension provider always extends `ServiceProvider`.

## Recommended `app/` layout

```
app/
├── Controllers/         # HTTP entry points (+ Traits/ for shared controller logic)
├── DTO/                 # request validation objects (group by domain)
├── Services/            # business orchestration (group by domain)
├── Repositories/
│   └── Contracts/       # repository interfaces (bound via 'alias')
├── Models/              # ORM models
├── Filters/             # QueryFilter classes for list endpoints
├── Events/              # domain events (BaseEvent)
├── Listeners/           # event subscribers (sync side effects)
├── Jobs/                # queued async work
├── Validation/Rules/    # custom validation rules
└── Providers/           # AppServiceProvider, EventServiceProvider
```

## Boundaries (the "where does it go?" rules)

- **Controllers stay thin** — no business logic, no SQL. If a controller method is doing real work, that work belongs in a service.
- **Validation lives in DTOs**, not in controllers (prevents per-endpoint validation drift).
- **Only repositories touch the database.** Services and controllers never build queries directly; they call repositories typed against contracts.
- **Services own use-cases and emit events**; they don't handle HTTP or know about requests/responses.
- **Side effects are events, not inline calls.** A service dispatches `ArticleCreated`; subscribers send notifications / bump counters / fire webhooks — and defer heavy work to jobs.
- **List endpoints use a Filter**, not hand-rolled `where()` chains in the controller.
- **Reach for an official extension** before building cross-cutting infrastructure (auth, RBAC, search, payments…).

## Implementation guides (per layer)

- Controllers → `glueful-add-controller` · Routes/docs → `glueful-add-route` · Data/queries → `glueful-build-orm-query` · Schema → `glueful-write-migration` · Tests → `glueful-write-test`
- Input validation → `glueful-build-validation-dto` · List filtering → `glueful-add-filter` · Side effects/async → `glueful-add-event-listener-job` · Packaging → `glueful-create-extension`

## Checklist (placing a new feature)

- [ ] Checked official extensions first; not duplicating package-provided functionality.
- [ ] HTTP only in the controller; input validated in a DTO.
- [ ] Business logic in a service; all data access in a repository behind a contract.
- [ ] Repository bound to its interface via `'alias'` in `AppServiceProvider::services()`; provider listed in `config/serviceproviders.php`.
- [ ] Side effects dispatched as domain events; subscribers wired via `EventService::subscribe()`; heavy work in jobs.
- [ ] List endpoints use a Filter for filtering/sorting/search.
