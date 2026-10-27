---
name: glueful-write-test
description: Write PHPUnit tests for Glueful — distinguishing consumer-app tests (extend App\Tests\TestCase / Glueful\Testing\TestCase, boot the real app per test) from framework/extension library tests (extend PHPUnit\Framework\TestCase, lightweight SQLite Connection harness). Covers the PHPUnit config an app needs, HTTP requests through the kernel, permission helpers, database setup, model factories, mocking, and the TDD loop. Use when adding or fixing tests in a Glueful app or package. Uses PHPUnit 10, not Laravel's TestCase/RefreshDatabase.
---

# Writing Glueful Tests

Glueful uses **PHPUnit 10** (attributes: `#[CoversClass]`, `#[Test]`, `#[DataProvider]` with `public static` providers) directly — no Laravel `TestCase`, no `RefreshDatabase`, no `artisan test`. **First decide which kind of test you're writing**, because the base class and bootstrap differ:

| | Consumer app (api-skeleton, your app) | Framework / extension library |
| --- | --- | --- |
| Base class | `App\Tests\TestCase` → `Glueful\Testing\TestCase` (or plain PHPUnit for pure unit tests) | `PHPUnit\Framework\TestCase` |
| Namespace | `App\Tests\Feature\...` / `App\Tests\Unit\...` | `Glueful\Tests\...` (or the package's own) |
| Bootstrap | Boots the **real app** fresh in every `setUp()` | None — build a `Connection`/`ApplicationContext` yourself |
| Resolve services | `$this->get(Foo::class)`, `$this->app()` | `createMock(...)` / construct directly |

Scaffold with `php glueful scaffold:test WidgetServiceTest` (unit, `--class=`/`--methods=`) or `--feature` (extends `App\Tests\TestCase` when `tests/TestCase.php` exists).

## Consumer app tests (the common case)

The api-skeleton ships `tests/TestCase.php`:

```php
namespace App\Tests;

use Glueful\Framework;
use Glueful\Testing\TestCase as FrameworkTestCase;

abstract class TestCase extends FrameworkTestCase
{
    protected function createApplication(): \Glueful\Application
    {
        return Framework::create(__DIR__ . '/..')->withEnvironment('testing')->boot();
    }
}
```

`Glueful\Testing\TestCase` gives you `app()` (the `Glueful\Application`), `getContainer()`, `get(string $id)`, `has(string $id)`, `refreshApplication()`, `createApplication()`/`getBasePath()` (override points — the default base path is wrong for apps, hence the override above), plus permission helpers. There is no `$context` in a test method — use `$this->app()->getContext()`.

```php
namespace App\Tests\Feature;

use App\Tests\TestCase;
use Symfony\Component\HttpFoundation\Request;

final class WidgetTest extends TestCase
{
    public function testListEndpoint(): void
    {
        $response = $this->app()->handle(Request::create('/widgets', 'GET'));   // full router + middleware pipeline
        self::assertSame(200, $response->getStatusCode());
        $payload = json_decode((string) $response->getContent(), true);
        self::assertTrue($payload['success'] ?? false);
    }

    public function testEditorCanPublish(): void
    {
        $uuid = $this->actingWithPermissions(['widgets.publish']);   // in-memory provider, ONLY $uuid granted
        self::assertTrue($this->get('permission.manager')->can($uuid, 'widgets.publish', 'widgets'));
    }
}
```

`actingWithPermissions()` / `actingWithRoles()` (roles from the `PermissionRegistry`; unknown role throws) install `Glueful\Testing\InMemoryPermissionProvider` and return the uuid — they don't authenticate a request. Framework API routes are under the API prefix (`/api/v1/...`); app route files control their own prefix. For user-store seams, swap in a test double (the skeleton's `tests/Support/InMemoryUserProvider`) rather than needing a DB.

### PHPUnit config & environment

The current skeleton ships a `phpunit.xml` (suites `Feature` + `Unit`); older apps may lack one — add it. Use the env names `config/database.php` actually reads: **`DB_DRIVER`** + **`DB_SQLITE_DATABASE`** (not `DB_CONNECTION`, not `DB_DATABASE`):

```xml
<php>
  <env name="APP_ENV" value="testing"/>
  <env name="CACHE_DRIVER" value="array"/>            <!-- avoids "Class Redis not found" on boot -->
  <env name="DB_DRIVER" value="sqlite"/>
  <env name="DB_SQLITE_DATABASE" value=":memory:"/>   <!-- or a file path for a migrated schema -->
  <env name="LOG_TO_FILE" value="false"/>
  <env name="LOG_TO_DB" value="false"/>
  <env name="JWT_KEY" value="testing-secret-key-not-for-production-use-only"/>
  <env name="APP_KEY" value="base64:…32 bytes…"/>
</php>
```

Run one suite with `composer test:unit` or `composer test:feature`.

### Database in app tests

- `:memory:` is a real in-memory DB (since 1.91) but **starts empty on every boot**, and `migrate:run` in another process can't fill it. Create the tables a test needs in `setUp()` via `$this->get(\Glueful\Database\Connection::class)->getSchemaBuilder()->createTable(...)`.
- For a migrated schema use a file: `DB_SQLITE_DATABASE=storage/database/testing.sqlite`, migrated once with `DB_DRIVER=sqlite DB_SQLITE_DATABASE=storage/database/testing.sqlite php glueful migrate:run --force` (real env vars beat `.env`; without `--force` a non-interactive run cancels).
- **No per-test transaction rollback:** `Connection` has no `beginTransaction()`/`rollback()` (only `transaction(callable)`, `afterCommit()`, `withinTransaction()`). Delete what you created in `tearDown()` or use unique values per test.

### Model factories & seeders

Factories need `composer require --dev fakerphp/faker` and the `Glueful\Database\ORM\Concerns\HasFactory` trait on the model. Generate with `php glueful scaffold:factory WidgetFactory` (`--model=` if not inferable) → `app/Database/Factories/WidgetFactory.php` (`App\Database\Factories`, extends `Glueful\Database\Factory\Factory`, `protected string $model`, `definition(): array` using `$this->faker`).

```php
$widget  = Widget::factory()->create(['name' => 'Alpha']);   // inside a booted app (uses the app's model context)
$widgets = Widget::factory(3)->createMany();                 // Collection; create() returns the first model
$draft   = Widget::factory()->state(['status' => 'draft'])->make();   // unsaved
Widget::factory(1, $context)->create();                      // outside a booted app: pass a context…
Widget::factory()->withContext($context)->create();          // …or set it explicitly
```

Seeders live in `app/Database/Seeders` (`App\Database\Seeders`, `scaffold:seeder`); `php glueful db:seed` runs `DatabaseSeeder`, `db:seed WidgetSeeder` one class.

## Framework / extension library tests

Tests extend `PHPUnit\Framework\TestCase` and assemble only what they need — no full boot. Layout mirrors the path (`tests/Unit/Database/WidgetTest.php` → `Glueful\Tests\Unit\Database`). Core tests must not depend on concrete extensions — use core contracts plus in-suite fakes.

### The SQLite `Connection` harness (DB/ORM tests)

```php
use Glueful\Database\Connection;

$this->connection = new Connection([
    'engine'  => 'sqlite',
    'sqlite'  => ['primary' => ':memory:'],      // or a temp file path; clean it up in tearDown()
    'pooling' => ['enabled' => false],
]);
$this->connection->getSchemaBuilder()->createTable('widgets', function ($t) {
    $t->id();
    $t->string('name', 50);
});
$this->connection->table('widgets')->insert(['name' => 'Alpha']);          // returns row count; insertGetId() for the id
$rows = $this->connection->table('widgets')->where('name', '=', 'Alpha')->get();
```

**ORM models without boot:** a model resolves its connection as the `database` service from its context's container. Build `new ApplicationContext($tmpDir, 'testing')`, `setContainer()` a tiny PSR-11 container returning the `Connection` for `'database'`, then use `Widget::query($context)` / `Widget::create($context, …)` (the framework's `tests/Unit/Database/ORM/Support/SqliteOrmHarness.php` is the reference).

### Lightweight helpers (no boot)

- **`ApplicationContext`** is path-constructible; `config($ctx, 'key', $default)` returns the default when no loader is wired.
- **Cache:** `new \Glueful\Cache\Drivers\ArrayCacheDriver()` (real in-memory TTL + tags).
- **`JWTService`** (static): set the key via reflection — `(new ReflectionClass(JWTService::class))->getProperty('key')->setValue(null, 'test-key');` (algorithm defaults to HS256).
- **Mocks:** core services like `TokenManager`/`NotificationService` are non-final → `createMock(...)`; capture args with `willReturnCallback`.

## TDD loop (expected by the codebase)

1. Write the failing test first. 2. Run it, confirm it fails for the right reason. 3. Minimal implementation. 4. Re-run green. 5. Run the surrounding suite before committing.

## Gotchas

- **Pick the right base class.** An app test on bare PHPUnit has no container; a library test on `Glueful\Testing\TestCase` tries to boot an app it doesn't have.
- **Routes load once per process.** Every test boots a new app, but `RouteManifest` loads once — call `\Glueful\Routing\RouteManifest::reset()` (and `\Glueful\Extensions\ServiceProvider::resetLoadedRoutes()` for extension route files) at the top of `createApplication()` when HTTP tests 404 after the first test.
- **Don't combine `Framework::boot()` with `#[RunInSeparateProcess]`** — the child's result serialization breaks.
- **`SoftDeleteHandler` caches "has `deleted_at`?" per `Connection` + table for the process** — give each hand-built schema its own `Connection`, or keep it identical to the real migration.
- **Prefer 3-arg `where('col', '=', $v)`**; a 2-arg string value that looks like an operator (`'like'`, `'is'`) is misread.
- **No Laravel test sugar:** no `RefreshDatabase`, `actingAs()`, `getJson()`, or `artisan`. HTTP = `$this->app()->handle(Request::create(...))`.

## Checklist

- [ ] Correct base class + namespace for the audience (app → `App\Tests\TestCase`; library → `PHPUnit\Framework\TestCase`).
- [ ] `phpunit.xml` defines the suites + testing env with `DB_DRIVER`/`DB_SQLITE_DATABASE`, `CACHE_DRIVER=array`, `JWT_KEY`.
- [ ] DB tests create their schema (`:memory:`) or run against a file migrated with `migrate:run --force`; cleanup in `tearDown()` (no transaction rollback).
- [ ] Factories: `fakerphp/faker` dev dep, `HasFactory` on the model, factory in `App\Database\Factories`; context passed when outside a booted app.
- [ ] Wrote the test failing first; ran the surrounding suite before committing.
