---
name: glueful-create-extension
description: Create or modify a Glueful framework extension — the composer manifest (type glueful-extension + extra.glueful.provider + the extra.glueful.migrations declaration), a ServiceProvider subclass, the static services()/defs() DI definitions, and the register()/boot() lifecycle (config merge, routes, commands, frontends, permission hooks). Use when building a packaged Glueful extension. Glueful extensions are NOT Laravel packages — DI is a static services() array, not $this->app->bind(), discovery is via composer extra.glueful, and schema is declared in the manifest, not loaded ad hoc.
---

# Creating a Glueful Extension

A Glueful extension is a Composer package whose service provider extends `Glueful\Extensions\ServiceProvider`. Scaffold one with **`php glueful create:extension <name>`** (verb-first — there is no `extensions:create` or `scaffold:extension`): it writes a full package under `extensions/<slug>/` (`composer.json` with `type: glueful-extension`, `extra.glueful.provider` and a migrations declaration; PSR-4 `Glueful\Extensions\<Name>\` → `src/`; `src/<Name>ServiceProvider.php`, `routes/routes.php`, `config/<slug>.php`, a flat `migrations/`), registers a Composer **path repository** in the app's `composer.json`, and **prints** `composer require glueful/<slug>:@dev && php glueful extensions:enable <slug>` (it does not run Composer). DI bindings are returned from a **static `services()` array**, not registered imperatively à la Laravel.

> **First, check the official catalog** (<https://glueful.com/extensions>). Build a custom extension only when no official one fits — user store (`glueful/users`), RBAC (`glueful/aegis`), multi-tenancy (`glueful/tenancy`), OAuth/SSO (`glueful/entrada`), audit log (`glueful/audit`), feature flags (`glueful/flags`), i18n (`glueful/i18n`), import/export (`glueful/import-export`), email (`glueful/email-notification`), push (`glueful/notiva`), SMS/WhatsApp (`glueful/conversa`), search (`glueful/meilisearch`), payments (`glueful/payvia`), commerce (`glueful/commerce`), subscriptions (`glueful/subscriptions`), runtime concurrency (`glueful/runiva`), image processing (`glueful/media`), CDN (`glueful/cdn`), queue supervision (`glueful/queue-ops`), archiving (`glueful/archive`), and cloud disks (`glueful/storage-s3|gcs|azure`) are already covered.

## 1. The composer manifest

The framework discovers extensions via `type` + `extra.glueful.provider`, and (since 1.80) reads their schema from `extra.glueful.migrations`:

```jsonc
{
  "name": "glueful/widgets",
  "type": "glueful-extension",
  "autoload": { "psr-4": { "Glueful\\Extensions\\Widgets\\": "src/" } },
  "require": { "php": "^8.3" },
  "extra": {
    "glueful": {
      "name": "Widgets",
      "displayName": "Widgets",
      "description": "...",
      "version": "1.0.0",
      "provider": "Glueful\\Extensions\\Widgets\\WidgetsServiceProvider",
      "requires": { "glueful": ">=1.91.0", "extensions": [] },
      "migrations": [
        { "id": "default", "path": "migrations", "priority": "default", "mode": "on_enable" }
      ]
    }
  }
}
```

- `extra.glueful.provider` (one FQCN) is the entry point — without it the extension isn't discovered.
- **`migrations` is mandatory.** A package with no declaration can't be enabled (`UndeclaredSchemaException`), and its provider may not register migration paths. A package with no tables declares `"migrations": "none"`. Each row: `id` (lowercase slug; `default` → ledger source = package name, else `package:id`), `path` (relative, inside the package — keep a `.gitkeep` so it exists), `priority` (`foundation|identity|platform|default|dependent`), `mode` (`on_enable` = run when the extension is enabled, skipped while disabled; `core` = always run), optional `verifier` (a `StructuralVerifierInterface` FQCN used by `migrate:verify --adopt`) and `previous_sources`.
- Tables that reference the user store use `"priority": "dependent"` so they run after identity/app migrations — and **no DB-level FK into `users`** (store an indexed `user_uuid`; resolve users via `Glueful\Auth\UserProviderInterface`).

> **Discovery ≠ activation.** An installed extension does nothing until its provider FQCN is in `config/extensions.php` → `enabled` (plain string FQCNs, no `::class`). Use `php glueful extensions:enable <name>`: it validates, **migrates the declared schema first**, then writes the list and recompiles — and is allowed in production. `requires.extensions` lists provider FQCNs that must **also** be enabled (not auto-enabled; an unmet dependency is refused). Providers in `config/extensions.php` → `protected` can't be toggled.

## 2. The service provider

```php
<?php

declare(strict_types=1);

namespace Glueful\Extensions\Widgets;

use Glueful\Bootstrap\ApplicationContext;
use Glueful\Extensions\ExtensionManager;
use Glueful\Extensions\ServiceProvider;

final class WidgetsServiceProvider extends ServiceProvider
{
    /** DI definitions, collected at container-compile time. Static — returns an array. */
    public static function services(): array
    {
        return [
            // Shared singleton, constructor args autowired from type hints:
            WidgetRepository::class => ['class' => WidgetRepository::class, 'shared' => true, 'autowire' => true],

            // Explicit arguments — '@' prefix references another service id:
            WidgetService::class => [
                'class' => WidgetService::class,
                'shared' => true,
                'arguments' => ['@' . WidgetRepository::class, '@database'],
            ],

            WidgetController::class => ['class' => WidgetController::class, 'shared' => true, 'autowire' => true],
        ];
    }

    /** Runs for every provider before any boot(). Config merging only. */
    public function register(ApplicationContext $context): void
    {
        $this->mergeConfig('widgets', require __DIR__ . '/../config/widgets.php');
    }

    /** Runs after ALL providers are registered. Routes, commands, frontends, listeners. */
    public function boot(ApplicationContext $context): void
    {
        $this->loadRoutesFrom(__DIR__ . '/../routes/routes.php');
        $this->discoverCommands(__NAMESPACE__ . '\\Console', __DIR__ . '/Console');
        // No loadMigrationsFrom(): migrations/ is registered from extra.glueful.migrations
        // and run by `extensions:enable` / `migrate:run`.

        // Metadata so extensions:list / extensions:info report it.
        $this->app->get(ExtensionManager::class)->registerMeta(self::class, [
            'slug' => 'widgets', 'name' => 'Widgets', 'version' => '1.0.0', 'description' => '...',
        ]);
    }
}
```

**Route files get `$router` only** — `$context` from `boot()` is not in scope. Inside `routes/routes.php` write `$context = $router->getContext();` before using `config($context, …)` etc.

### Service definition array shape (`services()`)

Each entry is `serviceId => array` (a non-array value throws):
- `'class'` — concrete class (inferred from a namespaced id if omitted).
- `'shared' => true` — singleton (the default); `false` for a fresh instance each time.
- `'autowire' => true` — resolve constructor params from their type hints.
- `'arguments' => [...]` — explicit constructor args; `'@id'` injects a service (core ids include `'@database'` for the connection and `'@cache.store'` for the cache — not `@db` / `@cache`). Object args are refused in production.
- `'factory' => [self::class, 'makeWidget']` (or `'Class::method'`, or `['@service', 'method']`) — called with the container. Closures are refused in production; use a static method.
- `'alias' => [WidgetServiceInterface::class]` — on the **concrete** entry, makes the interface id resolve to it (`Interface => ['alias' => Concrete]` does the opposite).
- `'tags' => ['some.tag']` — add to a tagged collection.

**Typed alternative — `static defs()`:** return `Glueful\Container\Definition\*` objects (`FactoryDefinition`, `AliasDefinition`, …). If a provider has both `defs()` and `services()`, **`defs()` wins and `services()` is ignored** — pick one.

## Lifecycle: `register()` → `boot()`

- `static services()` / `defs()` are collected at container-compile time (app providers first, then enabled extensions in resolved order).
- `register(ApplicationContext $context)` runs for **every** provider first — `mergeConfig()` here. App config always wins over your defaults, and a **list** value in the app's config **replaces** your list (since 1.86), it doesn't append.
- `boot(ApplicationContext $context)` runs **after all** providers are registered — routes, commands, `serveFrontend()`, event subscriptions (`$this->app->get(EventService::class)->subscribe(...)`), notification channels, and reads of other extensions' services.
- A provider that throws in `register()`/`boot()` is rethrown outside production (fail-fast) and logged-and-skipped in production; guard optional wiring (e.g. `$this->app->has(...)`) so a missing dependency doesn't take boot down.
- **Load order:** implement `Glueful\Extensions\DeclaresLoadOrder` (static `loadAfter(): array` of provider FQCNs + `loadPriority(): int`) when you must load after another provider; the one order applies to compilation, register, boot and the cache. Absent providers are ignored; cycles are errors. Don't combine with the legacy `OrderedProvider`.

## Activating an optional core capability (a "seam")

Core declares a contract and binds a **no-op default**; an extension *activates* the real behavior by binding its implementation to the **same id**. Extension definitions **replace** core definitions for an id, so you bind the interface — you don't reimplement the capability; uninstall and core falls back to the no-op.

| Core seam (contract) | Default with no extension | Bound by |
| --- | --- | --- |
| `Glueful\Uploader\Contracts\MediaProcessorInterface` | unbound → no-op upload path (no thumbnails, type-only metadata) | `glueful/media` |
| `Glueful\Cache\Contracts\EdgeCacheInterface` | `Glueful\Cache\NullEdgeCache` | `glueful/cdn` |
| `Glueful\Queue\Contracts\WorkerMonitorInterface` | `Glueful\Queue\Monitoring\NullWorkerMonitor` | `glueful/queue-ops` |
| `translation.manager` (string id) | unbound → `loadMessageCatalogs()` is a silent no-op | `glueful/i18n` |

A second seam shape is the **tagged collection**: the extension *adds* a tagged service that core iterates, so several extensions coexist. `glueful/storage-s3|gcs|azure` each register a `StorageDriverFactoryInterface` with `'tags' => ['storage.driver_factory']`, and core's `StorageDriverRegistryInterface` resolves `driver => s3|gcs|azure` disks through whichever are installed.

A plain autowirable impl can use the array form (`EdgeCacheInterface::class => ['class' => Impl::class, 'autowire' => true]`). When the impl needs config, use `defs()` with a `FactoryDefinition` (this is what `glueful/cdn` does):

```php
use Glueful\Bootstrap\ApplicationContext;
use Glueful\Cache\Contracts\EdgeCacheInterface;
use Glueful\Container\Definition\FactoryDefinition;
use Psr\Container\ContainerInterface;

public static function defs(): array   // NOT services() — services() entries must be arrays
{
    return [
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

## ServiceProvider helper methods (verified)

- `mergeConfig(string $key, array $defaults)` — merge default config under a key (call from `register()`). There is no `mergeConfigFrom()`.
- `loadRoutesFrom(string $path)` — require a route file with `$router` in scope (idempotent per file). Call from `boot()`.
- `discoverCommands(string $namespace, string $dir)` — register non-abstract `#[AsCommand]` classes under a dir; `commands(array $classes)` registers explicitly. Console only.
- `serveFrontend(string $path, string $dir, array $options = [])` — serve a prebuilt SPA/static bundle at a literal path (`'/admin/widgets'`: leading `/`, lowercase `[a-z0-9-]` segments, no trailing slash); options `spaFallback` (default true, needs `index.html`), `name`, `csp`. Replaces the removed `mountStatic()`.
- `loadMessageCatalogs(string $dir, string $domain = 'messages')` — register `messages.<locale>.php` catalogs (needs `glueful/i18n`).
- `registerNotificationChannel(NotificationChannel $channel)` / `registerNotificationExtension(NotificationExtension $ext)` — wire into the shared `ChannelManager` / `NotificationDispatcher` from `boot()`. This is the only channel wiring path; both no-op if notifications aren't in the container.
- `loadMigrationsFrom(string $dir, int $priority = MigrationPriority::DEFAULT, ?string $source = null, array $previousSources = [])` — **for app-local providers only.** Inside a package the manifest is the sole registrar: a path the manifest declares is a no-op (a contradicting source/priority throws `DescriptorValidationException`), an undeclared path throws, and an undeclared package throws `UndeclaredSchemaException`.
- **Authorization hooks** (instance methods, collected for every loaded provider): `permissions(): list<Permission>` (`Glueful\Permissions\Catalog\Permission::define('widgets.manage')->label(...)->category(...)->resource('widgets')->managedBy('glueful/widgets')`), `roles(): list<Role>` (`Role::define(...)->grants([...])`), `voters(): list<VoterInterface>`, `policies(): array<resource, class-string<PolicyInterface>>`.

## CLI

```bash
php glueful create:extension <name>          # scaffold package + path repository
php glueful extensions:list                  # enabled ✓ / available ○ / enabled-but-missing ⚠
php glueful extensions:info <name>
php glueful extensions:summary
php glueful extensions:enable <name>         # migrate declared schema, then enable + recompile (--dry-run, --backup)
php glueful extensions:disable <name>        # (--dry-run, --backup)
php glueful extensions:diagnose              # resolver errors, load order, prod cache presence
php glueful extensions:cache | extensions:clear
php glueful migrate:verify [--adopt=glueful/widgets]   # adopt pre-existing tables via the verifier
```

`enable`/`disable` accept a package name, provider FQCN, or slug (case-insensitive), validate before writing (missing dependency, version mismatch, or cycle leaves the config untouched), take the migration lock and record the operation. Commit the resulting `config/extensions.php` change.

## Not-Laravel reminders

- DI is a **static `services()` array** (or `defs()`), not `$this->app->bind()`/`singleton()` in `register()`.
- Discovery is **composer `extra.glueful.provider`**; activation is `config/extensions.php` → `enabled`.
- Schema is **declared in `extra.glueful.migrations`** and run by `extensions:enable` — no `vendor:publish`, no ad-hoc migration loading.
- `register()` is for config merge only; routes/commands/frontends/listeners go in `boot()`.

## Checklist

- [ ] `composer.json`: `type: "glueful-extension"`, PSR-4 autoload, `extra.glueful.provider`, `requires.glueful`, and `extra.glueful.migrations` (descriptor rows or `"none"`).
- [ ] Migrations in a flat `migrations/` at the package root (priority `dependent` if they reference users; no FK into `users`).
- [ ] Provider extends `Glueful\Extensions\ServiceProvider`; `register()`/`boot()` take `ApplicationContext $context`.
- [ ] DI via `static services(): array` (arrays only) or `static defs()` for `FactoryDefinition`s — not both.
- [ ] `register()` merges config; `boot()` loads routes/commands (+ `serveFrontend`, subscriptions) and calls `registerMeta()`; route files take `$context` from `$router->getContext()`.
- [ ] Enabled with `extensions:enable` and verified with `extensions:list` / `extensions:info` / `extensions:diagnose`.
