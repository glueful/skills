---
name: glueful-write-test
description: Write PHPUnit tests for Glueful — distinguishing consumer-app tests (extend App\Tests\TestCase / Glueful\Testing\TestCase, boot the real app) from framework/extension library tests (extend PHPUnit\Framework\TestCase, lightweight SQLite Connection harness). Covers the PHPUnit config an app needs, booting/mocking, and the TDD loop. Use when adding or fixing tests in a Glueful app or package. Uses PHPUnit 10, not Laravel's TestCase/RefreshDatabase.
---

# Writing Glueful Tests

Glueful uses **PHPUnit 10** (attributes: `#[CoversClass]`, `#[Test]`) directly — no Laravel `TestCase`, no `RefreshDatabase`, no `artisan test`. **First decide which kind of test you're writing**, because the base class and bootstrap differ:

| | Consumer app (api-skeleton, your app) | Framework / extension library |
| --- | --- | --- |
| Base class | `App\Tests\TestCase` → `Glueful\Testing\TestCase` | `PHPUnit\Framework\TestCase` |
| Namespace | `App\Tests\...` | `Glueful\Tests\...` (or the package's own) |
| Bootstrap | Boots the **real app** (`Framework::create()->boot()`) | None — build a lightweight `Connection`/`ApplicationContext` yourself |
| Resolve services | `$this->get(Foo::class)`, `$this->app()` | `createMock(...)` / construct directly |

## Consumer app tests (the common case)

App tests extend a local base that boots the framework with the `testing` environment:

```php
// tests/TestCase.php
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

```php
// tests/Feature/WidgetTest.php
namespace App\Tests\Feature;

use App\Tests\TestCase;

final class WidgetTest extends TestCase
{
    public function testServiceResolves(): void
    {
        $service = $this->get(\App\Services\WidgetService::class);   // from the booted container
        $this->assertNotNull($service);
    }
}
```

`Glueful\Testing\TestCase` gives you `createApplication()` (override point), `app()`, `getContainer()`, `get(string $id)`, `has(string $id)`, and `refreshApplication()` — all backed by a really-booted app.

### App skeletons need a PHPUnit config — add one before running

Some app skeletons ship **no `phpunit.xml`**, yet their composer scripts reference `--testsuite Unit|Integration`. So `composer test` / `vendor/bin/phpunit tests/Feature/...` will fail (undefined suites) or hit boot-time dependency errors (e.g. `Class "Redis" not found`) until you add a config that defines the suites **and** forces a test-safe environment.

**Use the env var names the app's own `config/database.php` actually reads — don't assume.** The standard Glueful config selects the engine via `DB_DRIVER` and the SQLite path via `DB_SQLITE_DATABASE` (it does **not** read `DB_CONNECTION`, and `DB_DATABASE` is only the *MySQL* database name). Inspect `config/database.php` first, then mirror those keys:

```xml
<phpunit bootstrap="vendor/autoload.php" colors="true" cacheDirectory=".phpunit.cache">
  <testsuites>
    <testsuite name="Unit"><directory>./tests/Unit</directory></testsuite>
    <testsuite name="Integration"><directory>./tests/Integration</directory></testsuite>
    <testsuite name="Feature"><directory>./tests/Feature</directory></testsuite>
  </testsuites>
  <php>
    <env name="APP_ENV" value="testing"/>
    <env name="CACHE_DRIVER" value="array"/>            <!-- avoids "Class Redis not found" on boot -->
    <env name="DB_DRIVER" value="sqlite"/>              <!-- engine selector — NOT DB_CONNECTION -->
    <env name="DB_SQLITE_DATABASE" value=":memory:"/>   <!-- sqlite path — NOT DB_DATABASE -->
    <env name="LOG_TO_FILE" value="false"/>
    <env name="LOG_TO_DB" value="false"/>
  </php>
</phpunit>
```

If feature/integration tests fail at boot: check (1) the config exists and defines the suites, (2) `CACHE_DRIVER=array` (Redis errors), and (3) the DB env keys match what `config/database.php` reads (`DB_DRIVER`/`DB_SQLITE_DATABASE`, not `DB_CONNECTION`/`DB_DATABASE`).

## Framework / extension library tests

When contributing to `glueful/framework` itself (or an extension package), tests extend `PHPUnit\Framework\TestCase` and you assemble only what you need — no full boot. Layout mirrors the path: `tests/Integration/Database/WidgetTest.php` → `namespace Glueful\Tests\Integration\Database;`. The framework ships its own `phpunit.xml` (with the testing env above) and `tests/bootstrap.php`.

### The SQLite `Connection` harness (DB/ORM tests)

Build a real `Connection` on a file-backed SQLite db with pooling off; create schema via PDO; clean up in `tearDown`:

```php
namespace Glueful\Tests\Integration\Database;

use Glueful\Database\Connection;
use PHPUnit\Framework\TestCase;

final class WidgetRepositoryTest extends TestCase
{
    private string $dbPath;
    private Connection $connection;

    protected function setUp(): void
    {
        parent::setUp();
        $this->dbPath = sys_get_temp_dir() . '/glueful-widget-' . uniqid('', true) . '.sqlite';
        $this->connection = new Connection([
            'engine'  => 'sqlite',
            'sqlite'  => ['primary' => $this->dbPath],
            'pooling' => ['enabled' => false],
        ]);
        $this->connection->getPDO()->exec('CREATE TABLE widgets (id INTEGER PRIMARY KEY, name TEXT)');
    }

    protected function tearDown(): void
    {
        if (is_file($this->dbPath)) { @unlink($this->dbPath); }
        parent::tearDown();
    }

    public function testReads(): void
    {
        $this->connection->getPDO()->exec("INSERT INTO widgets (name) VALUES ('Alpha')");
        $rows = $this->connection->table('widgets')->where('name', '=', 'Alpha')->get();  // 3-arg where!
        $this->assertCount(1, $rows);
    }
}
```

### Lightweight helpers (no boot)

- **`ApplicationContext`** is path-constructible; `config($ctx, 'key', $default)` returns the default when no loader is wired — deterministic config without booting:
  ```php
  $context = new \Glueful\Bootstrap\ApplicationContext(sys_get_temp_dir() . '/t-' . uniqid('', true));
  ```
- **Cache:** `new \Glueful\Cache\Drivers\ArrayCacheDriver()` (real in-memory TTL + tags; `remember`/`addTags`/`invalidateTags`).
- **`JWTService`** (static): set the key via reflection — `(new ReflectionClass(JWTService::class))->getProperty('key')->setValue(null, 'test-key');` (algorithm defaults to HS256).
- **Mocks:** core services like `TokenManager`/`NotificationService` are non-final → `createMock(...)`; capture args with `willReturnCallback`.

## TDD loop (expected by the codebase)

1. Write the failing test first. 2. Run it, confirm it fails for the right reason. 3. Minimal implementation. 4. Re-run green. 5. Run the surrounding suite before committing.

## Gotchas

- **Pick the right base class.** App test extending `PHPUnit\Framework\TestCase` won't have the booted container; a framework-internal test extending `Glueful\Testing\TestCase` would try to boot a full app it doesn't have. Match the table at the top.
- **App skeleton missing `phpunit.xml`** → add one (above) before running feature/integration tests; without `CACHE_DRIVER=array` you'll hit `Class "Redis" not found` at boot.
- **2-arg `where()`** treats the value as the operator — use the explicit 3-arg form `where('col', '=', $v)`.
- **No Laravel test sugar:** no `RefreshDatabase`, `$this->actingAs()`, `$this->getJson()` client, or `artisan` calls.
- **PHPStan analyzes `src`, not `tests`** — production code under test must pass; test files needn't.

## Checklist

- [ ] Chose the correct base class for the audience (app → `App\Tests\TestCase`; library → `PHPUnit\Framework\TestCase`) with the matching namespace.
- [ ] App project has a `phpunit.xml` defining the suites + `testing` env, using the DB env keys `config/database.php` actually reads (`DB_DRIVER`/`DB_SQLITE_DATABASE`, not `DB_CONNECTION`/`DB_DATABASE`) and `CACHE_DRIVER=array`.
- [ ] App tests resolve services via `$this->get(...)`/`$this->app()`; library tests build the SQLite `Connection` (pooling off) + clean up in `tearDown`.
- [ ] `where()` uses the explicit 3-arg form.
- [ ] Wrote the test failing first; ran the surrounding suite before committing.
