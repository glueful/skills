---
name: glueful-write-migration
description: Write a database migration for a Glueful framework project using the schema builder (create/alter/drop tables and columns, indexes, foreign keys) with idempotency guards and a working down() rollback. Use when adding or changing database schema in a Glueful app or extension. Glueful migrations are NOT Laravel migrations — they implement MigrationInterface and receive a SchemaBuilderInterface, not an Illuminate Blueprint.
---

# Writing a Glueful Migration

Glueful migrations are plain classes implementing `Glueful\Database\Migrations\MigrationInterface`. They are **not** Laravel migrations — there is no `Schema::create()` facade and no `Blueprint`; `up()` and `down()` receive a `SchemaBuilderInterface`. The table builder has Laravel-like helpers (`id()`, `timestamps()`, `softDeletes()`, `foreignId()`), but verify anything beyond that against `src/Database/Schema/Interfaces/`.

## Where migrations live & how to run them

App migrations live in the **consumer app's** `database/migrations/` (the api-skeleton ships it empty — core and extension schema come from their packages). Files are `NNN_ClassName.php`: the runner strips the `NNN_` prefix to get the class name, so `004_CreateWidgetsTable.php` must declare `CreateWidgetsTable` (a snake_case filename → "Migration class … not found").

```bash
php glueful migrate:create create_widgets_table   # snake_case arg → database/migrations/NNN_CreateWidgetsTable.php
php glueful migrate:run                # asks to confirm; default is NO
php glueful migrate:run --force        # required non-interactively (CI, deploys, scripts)
php glueful migrate:run --dry-run      # list what would run (alias --pretend)
php glueful migrate:rollback           # most recent migration; --steps=3 for more
php glueful migrate:status             # applied / pending, with source + description
php glueful migrate:verify             # per-source ready / adoptable / divergent (--adopt <pkg>, --json)
```

Without `--force`, a non-interactive `migrate:run` prints "Migration cancelled." and exits 0 **without migrating** — easy to miss in CI. `scaffold:model Widget --migration` also writes a runnable `NNN_Create<Table>Table.php`.

> **One ordered stream, many owners.** Framework core (`auth_sessions`, `api_keys`, queue, scheduler, notifications, locks, webhooks… — always installed since 1.79), extensions and the app all feed one stream sorted by **`MigrationPriority`** — `FOUNDATION` -200 → `IDENTITY` -100 → `PLATFORM` -50 → `DEFAULT` 0 → `DEPENDENT` 100 — then basename. The `migrations` table records a **`source`** (`app`, or the package name) so two owners can ship the same filename. App migrations are `DEFAULT`. **Don't add a DB-level foreign key into `users`** — the user store is a swappable extension (`glueful/users`); use an indexed `user_uuid` column with no `foreign()` constraint.
>
> **Extension migrations are declared in `composer.json`**, not registered in code (since 1.80): `"extra": {"glueful": {"migrations": [{"id": "default", "path": "migrations", "priority": "dependent", "mode": "on_enable"}]}}` (or `"migrations": "none"`). Priorities are `foundation | identity | platform | default | dependent`; `on_enable` lanes run when `extensions:enable` is called. Keep files in a flat `migrations/` dir at the package root and **don't** call `loadMigrationsFrom()` for a declared path (a contradicting call throws `DescriptorValidationException`; an undeclared package throws `UndeclaredSchemaException` on enable). `loadMigrationsFrom()` (from `boot()`) is only for app-local providers that belong to no package. See `glueful-create-extension`.

## The interface

```php
<?php

namespace Glueful\Database\Migrations;   // what migrate:create generates; the runner detects any namespace

use Glueful\Database\Migrations\MigrationInterface;
use Glueful\Database\Schema\Interfaces\SchemaBuilderInterface;

class CreateWidgetsTable implements MigrationInterface
{
    public function up(SchemaBuilderInterface $schema): void { /* ... */ }
    public function down(SchemaBuilderInterface $schema): void { /* ... */ }
    public function getDescription(): string
    {
        return 'Creates the widgets table.';   // stored in the ledger, shown by migrate:status
    }
}
```

All three methods are required. On PostgreSQL and SQLite each migration runs in a transaction with its ledger row.

## Creating a table — callback form (canonical)

`createTable(string $name, callable $callback)` builds and executes immediately. Guard with `hasTable()` so re-runs don't error:

```php
public function up(SchemaBuilderInterface $schema): void
{
    if ($schema->hasTable('widgets')) {
        return;
    }

    $schema->createTable('widgets', function ($table) {
        $table->id();                                    // auto-increment primary key `id`
        $table->string('uuid', 12)->unique();            // public identifier convention
        $table->string('name', 255);
        $table->text('description')->nullable();
        $table->boolean('is_active')->default(true);
        $table->integer('sort_order')->unsigned()->default(0);
        $table->string('user_uuid', 12);                 // no FK into users (see above)
        $table->foreignId('category_id')->constrained('categories')->nullOnDelete();
        $table->timestamps();                            // created_at (useCurrent) + updated_at (nullable)
        $table->softDeletes();                           // nullable deleted_at — honoured automatically

        $table->index('user_uuid');
        $table->index(['is_active', 'sort_order']);
    });
}

public function down(SchemaBuilderInterface $schema): void
{
    $schema->dropTableIfExists('widgets');               // executes immediately
}
```

### Column & constraint methods (verified)

- Columns: `id`, `string($n, $len = 255)`, `text`, `integer`, `bigInteger`, `boolean`, `decimal($n, 8, 2)`, `float`, `double`, `timestamp`, `dateTime`, `date`, `time`, `json`, `uuid`, `enum($n, $values)`, `binary`, `foreignId`, `addColumn($n, $type, $opts)`. Helpers: `timestamps()`, `softDeletes()`, `rememberToken()`. **No** `longText`/`smallInteger`/`tinyInteger`/`jsonb` — use `text`/`integer`/`json` or `addColumn()`.
- Modifiers: `nullable()`, `notNull()`, `default($v)`, `defaultRaw($expr)`, `useCurrent()`, `unsigned()`, `unique()`, `index()`, `primary()`, `autoIncrement()`, `check($expr)`, `comment()`, `constrained($table, 'id')`, `cascadeOnDelete()`/`nullOnDelete()`/`restrictOnDelete()`. `useCurrentOnUpdate()` is **MySQL-only** — it throws `UnsupportedSchemaOperationException` on PostgreSQL/SQLite (since 1.91), so leave it out of portable migrations.
- Table-level: `index($cols, ?$name)`, `unique($cols, ?$name)`, `primary($cols)`, `fulltext()`, and `foreign('col')->references('uuid')->on('posts')->nullOnDelete()` for non-`foreignId` keys.

## Altering a table — callback form

`alterTable($name, callback)` applies and executes immediately (it handles column-builder finalization internally — no manual `gc_collect_cycles()`/`execute()` needed):

```php
public function up(SchemaBuilderInterface $schema): void
{
    if ($schema->hasColumn('widgets', 'archived_at')) {
        return;   // idempotent — safe after a manual ALTER
    }
    $schema->alterTable('widgets', function ($table) {
        $table->timestamp('archived_at')->nullable();
        $table->index('archived_at', 'idx_widgets_archived_at');
    });
}

public function down(SchemaBuilderInterface $schema): void
{
    if (!$schema->hasColumn('widgets', 'archived_at')) {
        return;
    }
    $schema->alterTable('widgets', function ($table) {
        $table->dropIndex('idx_widgets_archived_at');
        $table->dropColumn('archived_at');
    });
}
```

Also inside `alterTable`: `renameColumn($from, $to)`, `modifyColumn('title')->string(300)`, `dropUnique()`, `dropForeign()`, `rename($new)`. Prefer this over the schema-level `$schema->dropColumn()`/`addColumn()` helpers, which return a result array instead of throwing and, on 1.91.2 and earlier, don't flush their SQL on MySQL/PostgreSQL. Likewise use `dropTableIfExists()` in `down()`: plain `dropTable()` only queues, and runners up to 1.91.2 never flush it. **SQLite** can't natively modify/drop columns or change FKs; Glueful does an audited create-copy-swap rebuild per `alterTable()` call and throws `UnsupportedSchemaOperationException` *before* changing anything if it can't preserve the table.

**Fluent (no-callback) form:** `table()`/`createTable($n)`/`alterTable($n)` return the table builder; column methods return a `ColumnBuilder`, so call **`->end()`** before the next column (chaining `->string('x')` onto a column builder is a `TypeError`), then `->create()`/`->execute()` and `$schema->execute()`. Use it mainly for `$schema->preview()` / `validate()` before running.

## Rules

1. **Make migrations idempotent.** Guard `up()` with `hasTable()`/`hasColumn()` and `down()` with the negation.
2. **Always implement a real `down()`** that reverses `up()` (drop dependent tables/constraints first).
3. **Use the schema builder, not raw SQL** — it generates driver-portable SQL (MySQL/PostgreSQL/SQLite).
4. **Never edit an applied migration** — checksums detect it; ship a follow-up migration.
5. **Test `migrate:run --force` then `migrate:rollback`** on SQLite at minimum (a scratch DB, e.g. `DB_DRIVER=sqlite DB_SQLITE_DATABASE=storage/database/scratch.sqlite`), ideally every target driver.

## Checklist

- [ ] File is `NNN_ClassName.php` with a matching class name (use `migrate:create`), implementing `up()`, `down()`, `getDescription()`.
- [ ] `up()` guarded by `hasTable()`/`hasColumn()`; `down()` guarded by the negation and actually reverses `up()`.
- [ ] Callback forms of `createTable`/`alterTable`; fluent chains (if any) use `->end()`.
- [ ] No raw SQL, no `useCurrentOnUpdate()` in portable migrations, no FK into `users`.
- [ ] Extension schema declared in `composer.json` `extra.glueful.migrations` with the right priority (no `loadMigrationsFrom()` for that path).
- [ ] `migrate:run --force` and `migrate:rollback` both verified clean.
