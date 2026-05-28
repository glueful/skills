---
name: glueful-write-test
description: Write PHPUnit tests for Glueful framework code — unit and integration tests, the SQLite-backed Connection harness for database/ORM tests, and patterns for mocking framework services (TokenManager, NotificationService), the cache, JWTService, and ApplicationContext. Use when adding or fixing tests in a Glueful framework or extension project. Uses PHPUnit 10 attributes, not Laravel's TestCase/RefreshDatabase.
---

# Writing Glueful Tests

Glueful uses **PHPUnit 10** (with attributes, `#[CoversClass]`/`#[Test]`) directly — there is no Laravel `TestCase`, no `RefreshDatabase`, no `artisan test`. Tests extend `PHPUnit\Framework\TestCase`.

## Layout, namespaces, commands

```
tests/Unit/         # pure logic, no I/O — testsuite "Unit"
tests/Integration/  # DB / cache / multi-component — testsuite "Integration"
tests/Feature/      # HTTP-level — testsuite "Feature"
```

Autoload: `Glueful\Tests\` → `tests/`. A test class mirrors its path:
`tests/Integration/Database/WidgetRepositoryTest.php` → `namespace Glueful\Tests\Integration\Database;`.

```bash
composer test                                   # all suites
composer run test:unit                          # --testsuite Unit
composer run test:integration                   # --testsuite Integration
vendor/bin/phpunit --filter="testMethodName"    # single test
vendor/bin/phpunit tests/Integration/Database/WidgetRepositoryTest.php
```

The suite runs with `APP_ENV=testing`, `CACHE_DRIVER=array`, `DB_CONNECTION=sqlite`, `DB_DATABASE=:memory:` (see `phpunit.xml`).

## Database / ORM integration tests — the SQLite Connection harness

The reusable pattern: build a real `Connection` on a file-backed SQLite db with pooling off, create the schema via PDO, and clean up in `tearDown`. This is how the framework's own DB integration tests work.

```php
<?php

declare(strict_types=1);

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

        $this->connection->getPDO()->exec(
            'CREATE TABLE widgets (id INTEGER PRIMARY KEY AUTOINCREMENT, uuid TEXT, name TEXT)'
        );
        $this->connection->getPDO()->exec("INSERT INTO widgets (uuid, name) VALUES ('w1', 'Alpha')");
    }

    protected function tearDown(): void
    {
        if (is_file($this->dbPath)) {
            @unlink($this->dbPath);
        }
        parent::tearDown();
    }

    public function testQueryBuilderReadsRows(): void
    {
        $rows = $this->connection->table('widgets')
            ->where('name', '=', 'Alpha')   // explicit 3-arg form — see gotcha below
            ->get();

        $this->assertCount(1, $rows);
        $this->assertSame('w1', $rows[0]['uuid']);
    }
}
```

> Use a unique temp path per test (`uniqid`) so parallel/repeat runs don't collide, and unlink in `tearDown`. `:memory:` also works but a file-backed db survives across multiple `Connection`/PDO handles in the same test, which you sometimes need.

## ApplicationContext without a full boot

Most framework code needs an `ApplicationContext`. It's path-constructible — and `config()` returns the supplied default when no config loader is wired, so you get deterministic values without booting the framework:

```php
use Glueful\Bootstrap\ApplicationContext;

$context = new ApplicationContext(sys_get_temp_dir() . '/glueful-test-' . uniqid('', true));
// config($context, 'security.auth.allowed_login_statuses', ['active']) === ['active']
```

For tests that need real config/services, boot via `Glueful\Framework::create($appPath)->boot(allowReboot: true)` and pull from the container — but prefer the lightweight context above when you only need config defaults.

## Cache-dependent tests

Use the in-memory `ArrayCacheDriver` (implements `CacheStore`, real TTL + tag support — `remember`/`addTags`/`invalidateTags`), no Redis needed:

```php
use Glueful\Cache\Drivers\ArrayCacheDriver;

$cache = new ArrayCacheDriver();
$cache->set('k', 'v', 300);
$cache->invalidateTags(['some-tag']);
```

## JWTService in tests

`JWTService` is static and reads a configured key. Set it directly via reflection (the framework's own tests do this); the algorithm defaults to HS256:

```php
use Glueful\Auth\JWTService;
use ReflectionClass;

(new ReflectionClass(JWTService::class))->getProperty('key')->setValue(null, 'test-key');
$token = JWTService::generate(['sub' => 'u1', 'sid' => 's1', 'ver' => 1], 3600);
```

## Mocking framework services

Core services like `TokenManager` and `NotificationService` are **non-final**, so `createMock()` works. Capture call args via `willReturnCallback` to assert on them:

```php
$tokenManager = $this->createMock(\Glueful\Auth\TokenManager::class);
$tokenManager->method('createUserSession')->willReturnCallback(
    function (array $user, ?string $provider = null): array {
        // record + return a fake OIDC session payload
        return ['access_token' => '…', 'user' => $user];
    }
);
```

## TDD loop (expected by the codebase)

1. Write the failing test first.
2. Run it, confirm it fails for the right reason (`vendor/bin/phpunit --filter=...`).
3. Implement the minimal code to pass.
4. Re-run; confirm green.
5. Run the surrounding suite (`composer run test:integration`) to catch regressions before committing.

## Gotchas

- **2-arg `where()`**: the shorthand only normalizes a non-string operand, so `where('status', 'active')` treats `'active'` as the *operator*. Use the explicit 3-arg form `where('status', '=', 'active')` in tests (and code).
- **PHPStan on tests**: the project analyzes `src` (not `tests`), so test files don't need to pass the strict level — but production code under test does.
- **No Laravel test sugar**: no `RefreshDatabase`/`DatabaseTransactions` traits, no `$this->actingAs()`, no `$this->getJson()` Laravel test client, no `artisan` calls. Build the `Connection`/context yourself as above.

## Checklist

- [ ] Test class under the right `tests/{Unit,Integration,Feature}/...` path with the matching `Glueful\Tests\...` namespace.
- [ ] DB tests use the file-backed SQLite `Connection` (pooling off) + schema in `setUp` + `unlink` in `tearDown`.
- [ ] `ApplicationContext` constructed lightweight (path only) unless real services are needed.
- [ ] Cache via `ArrayCacheDriver`; `JWTService` key set via reflection; non-final services mocked with `createMock`.
- [ ] `where()` uses the explicit 3-arg form.
- [ ] Wrote the test failing first, then made it pass; ran the surrounding suite before committing.
