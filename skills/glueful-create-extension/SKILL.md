---
name: glueful-create-extension
description: Create or modify a Glueful framework extension — the composer manifest (type glueful-extension + extra.glueful.provider), a ServiceProvider subclass, the static services() DI definitions, and register()/boot() lifecycle (routes, migrations, config merge, command discovery). Use when building a packaged Glueful extension. Glueful extensions are NOT Laravel packages — DI is a static services() array, not $this->app->bind(), and discovery is via composer extra.glueful, not a service-provider array.
---

# Creating a Glueful Extension

A Glueful extension is a Composer package whose service provider extends `Glueful\Extensions\ServiceProvider`. Scaffold one with **`php glueful create:extension <name>`** (note the namespace order — it's `create:extension`, **not** `extensions:create`), which writes the directory tree + a `ServiceProvider` stub; or build the manifest + provider by hand / copy an existing extension. DI bindings are returned from a **static `services()` array**, not registered imperatively à la Laravel.

> **First, check the official catalog** (<https://glueful.com/extensions>). Build a custom extension only when no official one fits — RBAC (`glueful/aegis`), OAuth/SSO (`glueful/entrada`), email (`glueful/email-notification`), push (`glueful/notiva`), search (`glueful/meilisearch`), payments (`glueful/payvia`), and runtime concurrency (`glueful/runiva`) are already covered.

## 1. The composer manifest

The framework discovers extensions via `type` + `extra.glueful.provider`:

```jsonc
{
  "name": "glueful/widgets",
  "type": "glueful-extension",
  "autoload": { "psr-4": { "Glueful\\Extensions\\Widgets\\": "src/" } },
  "require": { "glueful/framework": ">=1.46.0" },
  "extra": {
    "glueful": {
      "name": "Widgets",
      "displayName": "Widgets",
      "description": "...",
      "version": "1.0.0",
      "provider": "Glueful\\Extensions\\Widgets\\WidgetsServiceProvider",
      "requires": {
        "glueful": ">=1.46.0",
        "extensions": []
      }
    }
  }
}
```

The `extra.glueful.provider` FQCN is the entry point — without it the extension isn't discovered.

## 2. The service provider

```php
<?php

namespace Glueful\Extensions\Widgets;

use Glueful\Bootstrap\ApplicationContext;
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
        $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
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

## ServiceProvider helper methods (verified)

Available to your provider:
- `loadRoutesFrom(string $path)` — execute a route file (gets `$router` in scope).
- `loadMigrationsFrom(string $dir)` — register a migrations directory.
- `mergeConfig(string $key, array $defaults)` — merge extension config under a config key.
- `discoverCommands(string $namespace, string $dir)` — auto-register `#[AsCommand]` classes under a dir.
- `commands(array $classes)` — register CLI command classes explicitly.
- `mountStatic(string $mount, string $dir)` — serve static assets at `/extensions/{mount}/`.
- `loadMessageCatalogs(string $dir, string $domain = 'messages')` — register translation catalogs.

## CLI

Scaffold and manage extensions. **Mind the namespaces** — scaffolding is `create:extension` (verb-first), management is `extensions:*` (noun-first); there is no `extensions:create`.

```bash
php glueful create:extension <name>   # scaffold dir tree + ServiceProvider stub
php glueful extensions:list           # installed extensions
php glueful extensions:info <name>    # details
php glueful extensions:enable <name>
php glueful extensions:disable <name>
php glueful extensions:diagnose       # health/registration checks
php glueful extensions:why <name>     # why an extension is (not) loaded
php glueful extensions:cache          # compile/cache extension metadata
php glueful extensions:clear          # clear that cache
```

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
