---
name: glueful-create-extension
description: Create or modify a Glueful framework extension — the composer manifest (type glueful-extension + extra.glueful.provider), a ServiceProvider subclass, the static services() DI definitions, and register()/boot() lifecycle (routes, migrations, config merge, command discovery). Use when building a packaged Glueful extension. Glueful extensions are NOT Laravel packages — DI is a static services() array, not $this->app->bind(), and discovery is via composer extra.glueful, not a service-provider array.
---

# Creating a Glueful Extension

A Glueful extension is a Composer package whose service provider extends `Glueful\Extensions\ServiceProvider`. Scaffold one with **`php glueful create:extension <name>`** (note the namespace order — it's `create:extension`, **not** `extensions:create`): it writes a full Composer package under `extensions/<slug>/` (a `composer.json` with `type: glueful-extension` + `extra.glueful.provider`, PSR-4, `src/`, `routes/`, `config/`, `database/migrations/`), registers a Composer **path repository** in the app's `composer.json`, and **prints** the `composer require … && php glueful extensions:enable …` commands to finish (it does not run Composer itself). Or build the manifest + provider by hand / copy an existing extension. DI bindings are returned from a **static `services()` array**, not registered imperatively à la Laravel.

> **First, check the official catalog** (<https://glueful.com/extensions>). Build a custom extension only when no official one fits — the user store / identity & accounts (`glueful/users` — the first-party `UserProviderInterface`), RBAC (`glueful/aegis`), row-level multi-tenancy (`glueful/tenancy`), OAuth/SSO (`glueful/entrada`), email (`glueful/email-notification`), push (`glueful/notiva`), SMS/WhatsApp messaging (`glueful/conversa`), search (`glueful/meilisearch`), payments (`glueful/payvia`), runtime concurrency (`glueful/runiva`), image processing (`glueful/media`), edge cache / CDN (`glueful/cdn`), queue supervision / autoscaling (`glueful/queue-ops`), and table archiving (`glueful/archive`) are already covered.

## 1. The composer manifest

The framework discovers extensions via `type` + `extra.glueful.provider`:

```jsonc
{
  "name": "glueful/widgets",
  "type": "glueful-extension",
  "autoload": { "psr-4": { "Glueful\\Extensions\\Widgets\\": "src/" } },
  "require": { "glueful/framework": ">=1.52.0" },
  "extra": {
    "glueful": {
      "name": "Widgets",
      "displayName": "Widgets",
      "description": "...",
      "version": "1.0.0",
      "provider": "Glueful\\Extensions\\Widgets\\WidgetsServiceProvider",
      "requires": {
        "glueful": ">=1.52.0",
        "extensions": []
      }
    }
  }
}
```

The `extra.glueful.provider` FQCN is the entry point — without it the extension isn't discovered.

> **Discovery ≠ activation.** Composer *discovers* `glueful-extension` packages, but an
> installed extension does nothing until its provider FQCN is added to `config/extensions.php`'s
> single `enabled` allow-list (plain string FQCNs, no `::class`). Use `php glueful extensions:enable <name>`
> (it edits the list and recompiles), or add it by hand. `requires.extensions` lists provider FQCNs
> that must **also** be enabled — they aren't auto-enabled; enabling one with an unmet dependency is refused.

## 2. The service provider

```php
<?php

namespace Glueful\Extensions\Widgets;

use Glueful\Bootstrap\ApplicationContext;
use Glueful\Database\Migrations\MigrationPriority;
use Glueful\Extensions\ServiceProvider;

class WidgetsServiceProvider extends ServiceProvider
{
    /**
     * DI definitions for container compilation. Static — returns an array,
     * does NOT bind imperatively.
     */
    public static function services(): array
    {
        return [
            // Shared singleton, constructor args autowired from type hints:
            WidgetRepository::class => ['class' => WidgetRepository::class, 'shared' => true, 'autowire' => true],

            // Explicit arguments — '@' prefix references another service id:
            WidgetService::class => [
                'class' => WidgetService::class,
                'shared' => true,
                'arguments' => ['@' . WidgetRepository::class],
            ],

            WidgetController::class => [
                'class' => WidgetController::class,
                'shared' => true,
                'arguments' => ['@' . WidgetService::class],
            ],
        ];
    }

    /** Runtime registration — config merging. Runs for every provider before boot(). */
    public function register(ApplicationContext $context): void
    {
        $this->mergeConfig('widgets', require __DIR__ . '/../config/widgets.php');
    }

    /** Boot — runs after ALL providers are registered. Wire routes, migrations, commands. */
    public function boot(ApplicationContext $context): void
    {
        // Paths match the `create:extension` scaffold layout (see below):
        // provider in src/, routes in routes/, migrations in database/migrations/.
        $this->loadRoutesFrom(__DIR__ . '/../routes/routes.php');
        // Always pass the priority tier + a source tag (framework >= 1.50): DEFAULT is
        // right unless your tables must run after the user store (MigrationPriority::DEPENDENT).
        $this->loadMigrationsFrom(
            __DIR__ . '/../database/migrations',
            MigrationPriority::DEFAULT,
            'glueful/widgets',
        );
        $this->discoverCommands(__NAMESPACE__ . '\\Console', __DIR__ . '/Console');

        // Register metadata so extensions:list / extensions:info report it.
        $this->app->get(\Glueful\Extensions\ExtensionManager::class)->registerMeta(self::class, [
            'slug' => 'widgets',
            'name' => 'Widgets',
            'version' => '1.0.0',
            'description' => '...',
        ]);
    }
}
```

> **Scaffold layout** (what `php glueful create:extension Widgets` produces): `src/WidgetsServiceProvider.php`, `routes/routes.php`, `database/migrations/`, `config/`. Hence the `__DIR__ . '/../routes/routes.php'` and `__DIR__ . '/../database/migrations'` paths above — keep them aligned with the actual directory layout. Real extensions wrap each `boot()` step in try/catch (fail-fast in non-production) so one failure doesn't abort the whole framework boot.

### Service definition array shape

Each entry is `serviceId => definition`:
- `'class'` — concrete class to instantiate (usually same as the id).
- `'shared' => true` — singleton (resolved once); omit/false for a fresh instance each time.
- `'autowire' => true` — resolve constructor params from their type hints.
- `'arguments' => [...]` — explicit constructor args; prefix another service id with `'@'` to inject it (e.g. `'@' . WidgetRepository::class`). Mix with scalars/config values as needed.

## Lifecycle: `register()` → `boot()`

- `static services()` is collected at container-compile time.
- `register(ApplicationContext)` runs for **every** provider first — do config merging here (`mergeConfig`), not route/command wiring.
- `boot(ApplicationContext)` runs **after all** providers are registered — safe to load routes, register migrations, discover commands, and read other extensions' services.
- In real extensions, `boot()` wraps each step in try/catch and logs (fail-fast in non-production) so one extension's failure doesn't abort the whole boot.

## Activating an optional core capability (a "seam")

Framework 1.52+ ships several optional capabilities as a **seam**: core declares a contract and binds a **no-op default**, and an extension *activates* the real behavior by binding its implementation to the **same interface id** from `services()`. Providers load in order with core first, so the extension's binding for that id **wins** (last-provider-wins). You don't reimplement the capability — you bind the interface; without the extension, core degrades to the no-op.

| Core seam (contract) | Default with no extension | Bound by |
| --- | --- | --- |
| `Glueful\Uploader\Contracts\MediaProcessorInterface` | unbound → no-op upload path (no thumbnail, type-only metadata; `image()` undefined) | `glueful/media` |
| `Glueful\Cache\Contracts\EdgeCacheInterface` | `Glueful\Cache\NullEdgeCache` | `glueful/cdn` |
| `Glueful\Queue\Contracts\WorkerMonitorInterface` | `Glueful\Queue\Monitoring\NullWorkerMonitor` | `glueful/queue-ops` |
| `translation.manager` (string id) | unbound → every provider's `loadMessageCatalogs()` is a silent no-op | `glueful/i18n` |

A second seam shape is the **tagged collection**: instead of overriding one interface id, the extension *adds* a tagged service that core collects through an iterator — so multiple extensions coexist (additive) rather than last-provider-wins (exclusive). The storage registry works this way: `glueful/storage-s3|gcs|azure` each register a `StorageDriverFactoryInterface` under the `storage.driver_factory` tag, and core's `StorageDriverRegistryInterface` resolves disks with `driver => s3|gcs|azure` through whichever factories are installed.

```php
use Glueful\Bootstrap\ApplicationContext;
use Glueful\Container\Definition\FactoryDefinition;
use Glueful\Cache\Contracts\EdgeCacheInterface;
use Psr\Container\ContainerInterface;

public static function services(): array
{
    return [
        // Override core's no-op binding for this id — last-provider-wins.
        EdgeCacheInterface::class => new FactoryDefinition(
            EdgeCacheInterface::class,
            static function (ContainerInterface $c): EdgeCachePurger {
                $ctx = $c->get(ApplicationContext::class);
                return new EdgeCachePurger($ctx, (array) config($ctx, 'cdn', []));
            },
            true, // shared
        ),
    ];
}
```

A plain autowirable impl can use the array form (`EdgeCacheInterface::class => ['class' => Impl::class, 'shared' => true, 'autowire' => true]`); use a `FactoryDefinition` when the impl needs config or non-service constructor args — which the three extracted extensions do. This is how a lean core stays optional: install the extension to bind the real implementation, uninstall to fall back to the no-op default.

## ServiceProvider helper methods (verified)

Available to your provider:
- `loadRoutesFrom(string $path)` — execute a route file (gets `$router` in scope).
- `loadMigrationsFrom(string $dir, int $priority = MigrationPriority::DEFAULT, ?string $source = null)` — register a migrations directory. **Pass `$priority` and `$source` (framework ≥ 1.50):** `Glueful\Database\Migrations\MigrationPriority` tiers are `FOUNDATION` (-200), `IDENTITY` (-100, the `glueful/users` store), `DEFAULT` (0, the app), `DEPENDENT` (100). An extension whose tables reference the user store must register at `DEPENDENT` so it runs **after** identity/app migrations: `$this->loadMigrationsFrom(__DIR__.'/../migrations', MigrationPriority::DEPENDENT, 'glueful/yourext')`. The `$source` string tags the migrations in the `migrations` table's `source` column (two packages can ship the same filename without conflict; rollback resolves by `(source, migration)`).
- `mergeConfig(string $key, array $defaults)` — merge extension config under a config key.
- `discoverCommands(string $namespace, string $dir)` — auto-register `#[AsCommand]` classes under a dir.
- `commands(array $classes)` — register CLI command classes explicitly.
- `mountStatic(string $mount, string $dir)` — serve static assets at `/extensions/{mount}/`.
- `loadMessageCatalogs(string $dir, string $domain = 'messages')` — register translation catalogs.
- `registerNotificationChannel(NotificationChannel $channel)` — register a notification channel into the shared `ChannelManager` (framework ≥ 1.51). Call from `boot()`: `$this->registerNotificationChannel($this->app->get(MyChannel::class))`. This is now the **only** wiring path — the framework no longer hardcodes channel providers, so a channel not registered here won't reach the async dispatcher (queued/retried sends). Idempotent for the same class; throws `ChannelAlreadyRegisteredException` if a *different* class claims an already-registered name (`replaceChannel()` overrides intentionally). The channel implements `Glueful\Notifications\Contracts\NotificationChannel` (`send(): bool`), or `RichNotificationChannel` to return a structured `NotificationResult` (provider message id, error code, retryability, latency) from `sendNotification()` — the dispatcher prefers the rich path and falls back to `send()`.
- `registerNotificationExtension(NotificationExtension $ext)` — register before/after-send hooks on the shared `NotificationDispatcher` (framework ≥ 1.51). Call from `boot()`. Both helpers no-op if the notification subsystem isn't present in the container.

> **No cross-package foreign keys to `users` (framework ≥ 1.50).** The concrete user store lives in the `glueful/users` extension, so an extension's tables must **not** declare a DB-level FK into `users`. Store the principal id as an indexed UUID column (`user_uuid`) with no `foreign()` constraint, and resolve users through `Glueful\Auth\UserProviderInterface` in code — not a join you own. (This is what lets the user store be swapped or absent.)

## CLI

Scaffold and manage extensions. **Mind the namespaces** — scaffolding is `create:extension` (verb-first), management is `extensions:*` (noun-first); there is no `extensions:create`.

```bash
php glueful create:extension <name>   # scaffold dir tree + ServiceProvider stub
php glueful extensions:list           # installed extensions
php glueful extensions:info <name>    # details
php glueful extensions:enable <name>
php glueful extensions:disable <name>
php glueful extensions:diagnose       # resolver errors, load order, prod cache presence
php glueful extensions:cache          # compile the provider manifest (strict; required in prod)
php glueful extensions:clear          # clear that cache
```

`enable`/`disable` accept a package name, provider FQCN, or slug (case-insensitive),
**validate before writing** (refuse to leave the config broken), edit the `enabled`
list, and recompile. `extensions:list` shows each extension's state
(`enabled ✓` / `available ○` / `enabled-but-missing ⚠`).

`create:extension` gives you a working skeleton; fill in `services()`, routes, and migrations as above. Copying an existing extension under the org's `extensions/` is also a fast start.

## Not-Laravel reminders

- DI is a **static `services()` array**, not `$this->app->bind()`/`singleton()` in `register()`.
- Discovery is **composer `extra.glueful.provider`** (one FQCN), not a `providers` array in app config.
- No `php artisan make:provider`/`vendor:publish` — wire routes/migrations/config via the `ServiceProvider` helper methods in `boot()`/`register()`.
- `register()` is for config merge only; do route/migration/command wiring in `boot()`.

## Checklist

- [ ] `composer.json`: `type: "glueful-extension"`, PSR-4 autoload, and `extra.glueful.provider` set to the provider FQCN (+ `requires.glueful` version).
- [ ] Provider extends `Glueful\Extensions\ServiceProvider`.
- [ ] DI via `static services(): array` (class/shared/autowire/arguments with `@` refs) — not imperative binding.
- [ ] `register()` does config merge only; `boot()` loads routes/migrations/commands (guarded with try/catch) and calls `registerMeta()` so `extensions:list`/`info` report it.
- [ ] Route/migration paths match the scaffold layout (`../routes/routes.php`, `../database/migrations`).
- [ ] Verified with `extensions:list` / `extensions:info` / `extensions:diagnose` after enabling.
