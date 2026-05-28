---
name: glueful-write-migration
description: Write a database migration for a Glueful framework project using the schema builder (create/alter/drop tables and columns, indexes, foreign keys) with idempotency guards and a working down() rollback. Use when adding or changing database schema in a Glueful app. Glueful migrations are NOT Laravel migrations — they implement MigrationInterface and receive a SchemaBuilderInterface, not an Illuminate Blueprint.
---

# Writing a Glueful Migration

Glueful migrations are plain classes implementing `Glueful\Database\Migrations\MigrationInterface`. They are **not** Laravel migrations — there is no `Schema::create()` facade, no `Blueprint`, no `$table->id()`/`$table->timestamps()` sugar. You get a `SchemaBuilderInterface` passed into `up()` and `down()`.

## Where migrations live & how to run them

The framework ships only the migration *infrastructure* (`src/Database/Migrations/`). Actual migration files live in the **consumer app** (e.g. api-skeleton's `database/migrations/`), numbered sequentially: `001_CreateInitialSchema.php`, `010_AddTwoFactorEnabledToUsers.php`, etc.

```bash
php glueful migrate:create <name>   # scaffold a new migration
php glueful migrate:run             # apply pending migrations
php glueful migrate:rollback        # roll back the last batch
php glueful migrate:status          # show applied / pending
```

## The interface

```php
<?php

namespace Glueful\Database\Migrations;

use Glueful\Database\Migrations\MigrationInterface;
use Glueful\Database\Schema\Interfaces\SchemaBuilderInterface;

class CreateWidgetsTable implements MigrationInterface
{
    public function up(SchemaBuilderInterface $schema): void { /* ... */ }
    public function down(SchemaBuilderInterface $schema): void { /* ... */ }
    public function getDescription(): string
    {
        return 'Creates the widgets table.';
    }
}
```

All three methods are required. `up()`/`down()` return `void`; `getDescription()` returns a `string`.

## Creating a table — callback form (standard idiom)

`createTable(string $name, callable $callback)` builds and executes immediately. Guard with `hasTable()` so re-runs don't error:

```php
public function up(SchemaBuilderInterface $schema): void
{
    if ($schema->hasTable('widgets')) {
        return;
    }

    $schema->createTable('widgets', function ($table) {
        $table->bigInteger('id')->primary()->autoIncrement();
        $table->string('uuid', 12);
        $table->string('name', 255);
        $table->text('description')->nullable();
        $table->boolean('is_active')->notNull()->default(true);
        $table->integer('sort_order')->unsigned()->default(0);
        $table->timestamp('created_at')->default('CURRENT_TIMESTAMP');
        $table->timestamp('updated_at')->default('CURRENT_TIMESTAMP');
        $table->timestamp('deleted_at')->nullable();   // soft deletes

        $table->unique('uuid');
        $table->index('name');
    });
}

public function down(SchemaBuilderInterface $schema): void
{
    $schema->dropTableIfExists('widgets');
}
```

### Column & constraint methods (verified)

- Columns: `bigInteger`, `integer`, `string($name, $length)`, `text`, `boolean`, `timestamp`, `decimal`, etc.
- Modifiers (chained on the column): `->primary()`, `->autoIncrement()`, `->nullable()`, `->notNull()`, `->unsigned()`, `->default($value)`, `->defaultRaw($expr)`.
- Indexes (on the table): `->unique($col)`, `->index($col)`, foreign keys via the schema's foreign-key API.
- There is **no** Laravel `$table->id()` or `$table->timestamps()` shorthand — write the columns explicitly (the convention is a `bigInteger('id')->primary()->autoIncrement()` plus a short `string('uuid', 12)`).

## Altering an existing table — use the portable form

`alterTable()` has two forms. The **callback** form (`alterTable($name, function ($table) { ... })`) auto-executes but is a **newer** framework addition. To work across framework versions (including older pinned ones), prefer the **no-callback** form and flush explicitly:

```php
public function up(SchemaBuilderInterface $schema): void
{
    if ($schema->hasColumn('widgets', 'archived_at')) {
        return;   // idempotent — safe to re-run after a manual ALTER
    }

    $table = $schema->alterTable('widgets');
    $table->timestamp('archived_at')->nullable();

    // ColumnBuilder registers its column on __destruct; force collection
    // before the alteration SQL is generated.
    gc_collect_cycles();

    $table->execute();   // generate + queue the ALTER TABLE
    $schema->execute();  // flush pending SQL to the database
}

public function down(SchemaBuilderInterface $schema): void
{
    if (!$schema->hasColumn('widgets', 'archived_at')) {
        return;
    }
    $schema->dropColumn('widgets', 'archived_at');   // schema-level: alters + executes
}
```

> The `gc_collect_cycles()` + explicit `execute()` dance is required because the alter-mode column builder registers via `__destruct`. The framework's own `createTable` callback path does the same internally. On the callback form of `alterTable` you don't need it — but the no-callback form is the version-safe choice for shipped migrations.

## Rules

1. **Always make migrations idempotent.** Guard `up()` with `hasTable()`/`hasColumn()` and `down()` with the negation, so re-running after a partial/manual change doesn't error.
2. **Always implement a real `down()`.** It must reverse `up()` (drop the table/column you added). Don't leave it empty.
3. **Use the schema builder, not raw SQL.** Raw SQL bypasses the driver-portable generation (MySQL/PostgreSQL/SQLite).
4. **Test on SQLite at minimum**, ideally all three drivers — `migrate:run` then `migrate:rollback` should both succeed cleanly.
5. **Number the file sequentially** after the highest existing migration in the app's `database/migrations/` directory.

## Checklist

- [ ] Class implements `MigrationInterface` with `up()`, `down()`, `getDescription()`.
- [ ] `up()` guarded by `hasTable()`/`hasColumn()`; `down()` guarded by the negation.
- [ ] Columns/indexes written explicitly via the schema builder (no Laravel sugar, no raw SQL).
- [ ] `down()` actually reverses `up()`.
- [ ] For `alterTable`, used the no-callback form + `gc_collect_cycles()` + `execute()`/`$schema->execute()` for cross-version safety.
- [ ] `migrate:run` and `migrate:rollback` both verified clean.
