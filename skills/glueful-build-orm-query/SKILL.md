---
name: glueful-build-orm-query
description: Query and persist data with the Glueful ORM (Active Record) — Model statics, the query Builder, where constraints, eager loading with(), relations (hasOne/hasMany/belongsTo/belongsToMany/through), soft deletes, pagination, creating/updating models, transactions, query result caching, and avoiding N+1. Use when reading or writing model data in a Glueful app. The ORM looks Eloquent-ish but every entry point is context-first and it is NOT Laravel Eloquent.
---

# Building ORM Queries in Glueful

Glueful ships an Active Record ORM (`Glueful\Database\ORM\Model` + `Builder`). It resembles Eloquent in shape but **every entry point takes `ApplicationContext` first**, and it's its own implementation — verify signatures, don't assume Eloquent. Model queries run through the same `QueryBuilder` as `db($context)->table(...)`, so soft deletes, typed exceptions and connection handling behave identically.

## Getting a query / fetching

Model statics are **context-first** (in a controller the context is `$this->context`; in a service, inject `ApplicationContext`):

```php
User::query($context)                 // ORM Builder
    ->where('status', '=', 'active')  // explicit 3-arg form (see gotcha)
    ->orderBy('created_at', 'desc')   // or ->latest() / ->oldest()
    ->limit(50)
    ->get();                          // Glueful\Database\ORM\Collection

User::find($context, $id);            // ?User
User::findOrFail($context, $id);      // User | throws ModelNotFoundException (-> 404)
User::all($context);                  // Collection
User::with($context, ['profile', 'posts'])->get();
```

Builder methods (verified): terminals `get()`, `first()`, `firstOrFail()`, `find()`, `findMany()`, `findOrFail()`, `count()`, `exists()`, `paginate()`, `chunk($n, $cb)`, `each($cb)`; filters `where()`/`orWhere()`, `whereIn()`/`whereNotIn()`, `whereNull()`/`whereNotNull()`, `has()`/`doesntHave()`, `whereHas()`/`orWhereHas()`/`whereDoesntHave()`; shaping `with()`, `without()`, `withCount()`, `orderBy()`/`orderByDesc()`/`latest()`/`oldest()`, `limit()`/`take()`, `offset()`/`skip()`, `forPage()`, `join()`/`leftJoin()`, `groupBy()`. Anything else (`whereLike`, `whereBetween`, `whereJsonContains`, `selectRaw`, `pluck`, `max`, `cache`, …) is forwarded to the underlying `QueryBuilder`. There is **no** `sum()`/`avg()`/`min()` on the builder — use `selectRaw()`, or the Collection's `sum()`/`avg()`/`min()` on loaded rows.

**Pagination** returns an array whose `data` is a `Collection` of models, with `with()`/`withCount()` applied (since 1.91; earlier it was raw rows):

```php
$page = Post::query($context)->with('author')->paginate(1, 20);   // (page, perPage)
// ['data' => Collection<Post>, 'current_page' => 1, 'per_page' => 20, 'total' => …, 'last_page' => …, 'has_more' => …]
```

## Creating, updating, deleting

```php
$user = User::create($context, ['email' => 'a@b.com', 'status' => 'active']);
$user->id;                               // generated key read back

$user = new User(['email' => 'a@b.com'], $context);   // context is the 2nd ctor arg
$user->save();                           // bool

User::firstOrCreate($context, ['email' => 'a@b.com'], ['status' => 'active']);
User::updateOrCreate($context, ['email' => 'a@b.com'], ['status' => 'active']);

$user->status = 'disabled';
$user->save();                           // writes only dirty columns
$user->update(['status' => 'active']);   // fill + save
$user->delete();                         // bool
User::destroy($context, [1, 2, 3]);      // int (loads + deletes each)

// Bulk: returns affected-row count and bypasses model events
User::query($context)->where('status', '=', 'pending')->update(['status' => 'expired']);
```

Mass assignment: with a non-empty `protected array $fillable`, unlisted keys are **silently dropped**; with an empty `$fillable` and the default `$guarded = ['*']`, any key **throws**. `$timestamps` and `$incrementing` are **public** properties — redeclare as `public bool $timestamps = false;` (a `protected` redeclaration is fatal).

**Transactions** live on the connection, not the model: `db($context)->transaction(function () use ($context) { … })` commits on return and rolls back on throw (plus `afterCommit()`, `withinTransaction()`); there is no `beginTransaction()`. Write failures throw typed exceptions from `Glueful\Database\Exceptions` — catch e.g. `UniqueConstraintViolationException` (rendered as 409) instead of string-matching driver messages.

## Soft deletes

They follow the table: if it has a `deleted_at` column, `delete()` sets it and queries hide those rows automatically.

```php
Post::query($context)->withTrashed()->get();                         // include trashed
Post::query($context)->onlyTrashed()->get();                         // only trashed
Post::query($context)->where('id', '=', $id)->restore();
Post::query($context)->where('id', '=', $id)->forceDelete();         // real DELETE (since 1.91)
```

Add `use Glueful\Database\ORM\Concerns\SoftDeletes;` to the model for instance `trashed()`, `restore()`, `forceDelete()` and their events.

## Defining relations

```php
use Glueful\Database\ORM\Relations\{BelongsTo, BelongsToMany, HasMany, HasOne};

class User extends Model
{
    public function profile(): HasOne { return $this->hasOne(Profile::class); }   // (related, ?foreignKey, ?localKey)
    public function posts(): HasMany   { return $this->hasMany(Post::class); }
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)->withPivot('granted_at');      // pivot table role_user
    }
}

class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');   // (related, ?foreignKey, ?ownerKey, ?relation)
    }
}
```

Available: `hasOne`, `hasMany`, `belongsTo`, `belongsToMany`, `hasOneThrough`, `hasManyThrough`. `belongsTo` guesses its foreign key from the **method name** (`author()` → `author_id`), so pass it explicitly when the column is named after the related model. Pivot rows: `$user->roles()->attach($id, [...])`, `detach($ids)`, `sync([...])`, `toggle([...])`.

## Eager loading — always prefer it (N+1 safety)

```php
$users = User::with($context, ['posts', 'profile'])->get();
foreach ($users as $user) {
    echo $user->profile->bio;     // already loaded — no extra query
}

User::query($context)
    ->with(['posts.comments', 'posts' => fn ($q) => $q->where('published', '=', true)])
    ->withCount('posts')                                          // adds posts_count
    ->whereHas('posts', fn ($q) => $q->where('published', '=', true))
    ->get();

$user->load('posts'); $user->loadMissing('profile'); $user->loadCount('posts');
```

**N+1 detection is built in.** Lazy-loading a relation on a model hydrated from a multi-row result is flagged. Mode via `DB_LAZY_LOADING_MODE` (`config/database.php` → `orm.lazy_loading_mode`): `off | warn | strict | auto` (auto = warn when `APP_ENV=development`, else off); strict throws `LazyLoadingViolationException`. Per-model opt-out: `protected ?string $instanceLazyLoadingMode = 'off';`; enforce in tests with `Model::preventLazyLoading('strict')`. The right fix is almost always adding the relation to `with()`, not silencing the detector.

## Query result caching

`->cache(?int $ttl = null, array $tags = [])` is forwarded to the `QueryBuilder`, caching the read via `QueryCacheService` and tagging it by table plus your tags:

```php
User::query($context)->where('status', '=', 'active')->cache(ttl: 3600, tags: ['users'])->get();

$cache->invalidateTags(['users']);   // writes do NOT auto-invalidate; per-table tag is query_cache:table:users
```

**Caution:** the cache key is `query + params` with no auth/context scoping — never cache a user-scoped query without a user-discriminating clause in the SQL.

## Gotchas

- **Context-first everywhere.** `query()`, `find()`, `all()`, `create()`, `with()`, `destroy()` all take `$context` first; a contextless `User::find($id)` is a `TypeError` (Laravel reflex).
- **Prefer 3-arg `where()`.** The 2-arg form `where('status', 'active')` works for ordinary values, but a string value that looks like an operator (`'like'`, `'in'`, `'is'`) is taken as the operator, and a two-word value (`'on hold'`) throws "Invalid SQL operator". `where('col', '=', $v)` is always unambiguous.
- **`get()` returns a Glueful `Collection`** (`first`, `map`, `filter`, `where`, `pluck`, `keyBy`, `groupBy`, `sortBy`, `sum`, `toArray`, …) — check `src/Database/ORM/Collection.php` rather than assuming Laravel helpers.
- **Relations are read as properties** (`$user->posts`, cached) but **defined as methods**; calling `$user->posts()` returns the relation/query, not results.
- **Model events fire only for instance writes.** Returning `false` from a `saving`/`creating`/`updating`/`deleting` callback cancels the write (since 1.91); builder-level `update()`/`delete()` skip events.

## Checklist

- [ ] Used context-first statics (`Model::query($context)`, `find($context, …)`, `create($context, …)`).
- [ ] `where()` uses the explicit 3-arg form for string values.
- [ ] Relations needed in a loop are eager-loaded via `with()`; used `whereHas`/`withCount` for constraints/counts; `belongsTo` keys passed explicitly where the name differs.
- [ ] Writes go through `create()` / `new … + save()` / `updateOrCreate()`, respecting `$fillable`; multi-write units wrapped in `db($context)->transaction()`.
- [ ] No Laravel/nonexistent APIs (`sum/avg/min` on the builder, `beginTransaction()`, contextless statics).
- [ ] If caching, `->cache(ttl, tags)` with explicit invalidation, and the query is not user-scoped without a discriminator.
