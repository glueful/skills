---
name: glueful-build-orm-query
description: Query and persist data with the Glueful ORM (Active Record) — Model statics, the query Builder, where constraints, eager loading with(), relations (hasOne/hasMany/belongsTo/belongsToMany/through), pagination, creating/updating models, query result caching, and avoiding N+1. Use when reading or writing model data in a Glueful app. The ORM looks Eloquent-ish but every entry point is context-first and it is NOT Laravel Eloquent.
---

# Building ORM Queries in Glueful

Glueful ships an Active Record ORM (`Glueful\Database\ORM\Model` + `Builder`). It resembles Eloquent in shape but **every entry point takes `ApplicationContext` first**, and it's its own implementation — verify signatures, don't assume Eloquent.

## Getting a query / fetching

Model statics are **context-first**:

```php
use Glueful\Bootstrap\ApplicationContext;

User::query($context)                 // Builder
    ->where('status', '=', 'active')  // 3-arg form (see gotcha)
    ->orderBy('created_at', 'desc')
    ->limit(50)
    ->get();                          // Collection

User::find($context, $id);            // ?Model
User::findOrFail($context, $id);      // Model | throws
User::all($context);                  // Collection
User::with($context, ['profile', 'posts'])->get();
```

Builder terminals & filters (verified): `get(): Collection`, `first(): ?Model`, `find()`, `findMany()`, `findOrFail()`, `firstOrCreate()`, `firstOrNew()`, `where()`, `orWhere()`, `whereIn()`, `whereNotIn()`, `whereHas()`/`orWhereHas()`/`whereDoesntHave()`, `with()`, `without()`, `withCount()`, `orderBy()`, `limit()`, `offset()`. Pagination and other query-builder methods are proxied through `Builder::__call` to the underlying `QueryBuilder` (e.g. `->paginate(page: 1, perPage: 25)`).

`get()` returns a `Glueful\Database\ORM\Collection` (not a Laravel collection) of `Model` instances.

## Creating, updating, deleting

```php
// Static create (context-first) — yes, this exists
$user = User::create($context, ['email' => 'a@b.com', 'status' => 'active']);

// Instance build + save
$user = new User(['email' => 'a@b.com'], $context);
$user->save();          // bool

// Upserts
User::firstOrCreate($context, ['email' => 'a@b.com'], ['status' => 'active']);
User::updateOrCreate($context, ['email' => 'a@b.com'], ['status' => 'active']);

// Update / delete an instance
$user->status = 'disabled';
$user->save();
$user->delete();        // bool
```

Mass assignment respects the model's `protected array $fillable`.

## Defining relations

Declare relation methods on the model (verified signatures):

```php
class User extends Model
{
    public function profile(): \Glueful\Database\ORM\Relations\HasOne
    {
        return $this->hasOne(Profile::class);              // (related, ?foreignKey, ?localKey)
    }

    public function posts(): \Glueful\Database\ORM\Relations\HasMany
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function author(): \Glueful\Database\ORM\Relations\BelongsTo
    {
        return $this->belongsTo(User::class);              // (related, ?foreignKey, ?ownerKey, ?relation)
    }
}
```

Available: `hasOne`, `hasMany`, `belongsTo`, `belongsToMany`, `hasOneThrough`, `hasManyThrough`. Keys are inferred from convention; pass them explicitly when they differ.

## Eager loading — always prefer it (N+1 safety)

Load relations up front with `with()` and read them as properties:

```php
$users = User::with($context, ['posts', 'profile'])->get();
foreach ($users as $user) {
    echo $user->profile->bio;     // already loaded — no extra query
    foreach ($user->posts as $post) { /* ... */ }
}

// Nested + counts + constrained
User::query($context)
    ->with(['posts.comments'])
    ->withCount('posts')
    ->whereHas('posts', fn ($q) => $q->where('published', '=', true))
    ->get();
```

**N+1 detection is built in.** Accessing a relation that wasn't eager-loaded, on a member of a hydrated collection, is flagged. Modes via `DB_LAZY_LOADING_MODE` (or `config/database.php` → `orm.lazy_loading_mode`): `off | warn | strict | auto` (auto = warn in dev, off otherwise); strict throws `LazyLoadingViolationException`. Per-model opt-out: `protected ?string $instanceLazyLoadingMode = 'off';`. The right fix for a warning is almost always to add the relation to `with()`, not to silence the detector.

## Query result caching (framework ≥ 1.46.0)

`->cache(?int $ttl = null, array $tags = [])` proxies through the builder to `QueryCacheService`, caching the read and tagging it by table plus your tags:

```php
User::query($context)
    ->where('status', '=', 'active')
    ->cache(ttl: 3600, tags: ['users'])
    ->get();

// Invalidate on write:
$cache->invalidateTags(['users']);                 // your tag
// (automatic per-table tag query_cache:table:users is also set)
```

In framework < 1.46.0 `->cache()` is a no-op — check the version before relying on it. **Caution:** the cache key is `query + params` with no auth/context scoping — never cache a query that returns user-scoped rows without a user-discriminating clause in the SQL, or you risk serving one user's data to another.

## Gotchas

- **Context-first everywhere.** `Model::query()`, `find()`, `all()`, `create()`, `with()` all take `$context` as the first arg. A contextless call is a Laravel reflex and won't compile.
- **2-arg `where()`** only normalizes a non-string operand, so `where('status', 'active')` treats `'active'` as the operator. Use `where('status', '=', 'active')`.
- **`get()` returns a Glueful `Collection`**, not a Laravel one — check `src/Database/ORM/Collection.php` for its methods rather than assuming Laravel collection helpers.
- **Relations are read as properties** (`$user->posts`) but **defined as methods** returning relation objects; calling the method (`$user->posts()`) returns the relation/query, not the results.

## Checklist

- [ ] Used context-first statics (`Model::query($context)`, `find($context, …)`, `create($context, …)`).
- [ ] `where()` uses the explicit 3-arg form.
- [ ] Relations needed in a loop are eager-loaded via `with()` (no lazy-load N+1); used `whereHas`/`withCount` for constraints/counts.
- [ ] Writes go through `create()` / `new … + save()` / `updateOrCreate()`, respecting `$fillable`.
- [ ] If caching, `->cache(ttl, tags)` (framework ≥ 1.46) and the query is not user-scoped without a discriminator.
