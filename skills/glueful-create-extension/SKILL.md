---
name: glueful-create-extension
description: Create or modify a Glueful framework extension — the composer manifest (type glueful-extension + extra.glueful.provider), a ServiceProvider subclass, the static services() DI definitions, and register()/boot() lifecycle (routes, migrations, config merge, command discovery). Use when building a packaged Glueful extension. Glueful extensions are NOT Laravel packages — DI is a static services() array, not $this->app->bind(), and discovery is via composer extra.glueful, not a service-provider array.
---

# Creating a Glueful Extension

A Glueful extension is a Composer package whose service provider extends `Glueful\Extensions\ServiceProvider`. There is **no `php glueful extensions:create` scaffolder** — build the two pieces by hand (manifest + provider), or copy an existing extension's layout. DI bindings are returned from a **static `services()` array**, not registered imperatively à la Laravel.

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
        $this->loadRoutesFrom(__DIR__ . '/routes.php');
        $this->loadMigrationsFrom(dirname(__DIR__) . '/migrations');
        $this->discoverCommands(__NAMESPACE__ . '\\Console', __DIR__ . '/Console');
    }
}
```

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

## Managing extensions (CLI)

There is **no `extensions:create`**. The real commands:

```bash
php glueful extensions:list        # installed extensions
php glueful extensions:info <name> # details
php glueful extensions:enable <name>
php glueful extensions:disable <name>
php glueful extensions:diagnose    # health/registration checks
php glueful extensions:cache       # compile/cache extension metadata
php glueful extensions:clear       # clear that cache
```

To scaffold a new one: create the package directory with the composer manifest + provider above (copying an existing extension under the org's `extensions/` is the fastest start), then `composer require` / path-repo it into the app and `extensions:enable` it.

## Not-Laravel reminders

- DI is a **static `services()` array**, not `$this->app->bind()`/`singleton()` in `register()`.
- Discovery is **composer `extra.glueful.provider`** (one FQCN), not a `providers` array in app config.
- No `php artisan make:provider`/`vendor:publish` — wire routes/migrations/config via the `ServiceProvider` helper methods in `boot()`/`register()`.
- `register()` is for config merge only; do route/migration/command wiring in `boot()`.

## Checklist

- [ ] `composer.json`: `type: "glueful-extension"`, PSR-4 autoload, and `extra.glueful.provider` set to the provider FQCN (+ `requires.glueful` version).
- [ ] Provider extends `Glueful\Extensions\ServiceProvider`.
- [ ] DI via `static services(): array` (class/shared/autowire/arguments with `@` refs) — not imperative binding.
- [ ] `register()` does config merge only; `boot()` loads routes/migrations/commands (guarded with try/catch).
- [ ] Verified with `extensions:list` / `extensions:info` / `extensions:diagnose` after enabling.
