---
name: glueful-add-filter
description: Add filtering, sorting, and full-text search to a Glueful list endpoint with a QueryFilter class — declaring filterable/sortable/searchable fields, a default sort, custom filter{Field}() methods, and safe public filtering. Use when building or hardening a list/index endpoint in a Glueful app. Glueful's QueryFilter is request-driven (?filter[...]/?sort/?search) and applied to a QueryBuilder, not a Laravel scope or Spatie QueryBuilder.
---

# Adding a Query Filter

Glueful list endpoints get filtering/sorting/search from a **`QueryFilter`** subclass (`Glueful\Api\Filtering\QueryFilter`). It parses `?filter[...]`, `?search=`, and `?sort=` from the request and applies them to a `QueryBuilder`. You declare which fields are exposed; everything else is rejected. Applied in the **service** layer (per `glueful-architect-app`), not the controller.

## Define the filter

```php
namespace App\Filters;

use Glueful\Api\Filtering\QueryFilter;

class ArticleFilter extends QueryFilter
{
    /** Fields allowed for ?filter[...]. null = ALL fields filterable (see "safe public filtering"). */
    protected ?array $filterable = ['author_uuid', 'category', 'status', 'created_at'];

    /** Fields allowed for ?sort=. null = all sortable. */
    protected ?array $sortable = ['created_at', 'view_count', 'comment_count'];

    /** Fields included in ?search= (full-text OR across these). */
    protected array $searchable = ['title', 'body'];

    /** Default sort when the request specifies none. '-' prefix = descending. */
    protected ?string $defaultSort = '-created_at';
}
```

Scaffold a starting point with `php glueful scaffold:filter <Name>`.

## Request syntax (what users send)

```
GET /articles?filter[category]=news                          # equality
GET /articles?filter[category][in][]=news&filter[category][in][]=opinion   # IN
GET /articles?filter[view_count][gte]=100                    # operator (gte/lte/ne/...)
GET /articles?search=climate                                 # OR across $searchable
GET /articles?sort=-view_count                               # '-' = desc
GET /articles?filter[category]=news&sort=-created_at&search=energy   # combined
```

Operators are resolved through the framework's filter-operator layer; non-allowed fields are silently ignored (not applied), so the whitelist is the security boundary.

## Apply it (service layer)

Get the base query from the **repository** (data access stays there), apply the filter in the **service**, then paginate:

```php
// app/Services/Article/ArticleService.php
use App\Filters\ArticleFilter;

public function feed(Request $request, int $page, int $perPage): array
{
    $query  = $this->articleRepository->feedQuery();        // repository owns the base query (joins, alias 'a', visibility)
    $query  = (new ArticleFilter($request))->apply($query); // service applies request filter/sort/search
    return $query->paginate($page, $perPage);
}
```

> The repository builds the base `QueryBuilder` (joins, the `a` table alias, soft-delete/visibility scoping); the service only layers the request-driven filter on top. Keeps data-access construction in the repository (`glueful-architect-app`).

`apply(QueryBuilder $query): QueryBuilder` returns the same builder for chaining. The filter reads the request in its constructor — no middleware required for this pattern (the `FilterMiddleware` only pre-parses `_filters` into request attributes if you want that).

## Custom `filter{Field}()` methods

For anything beyond a plain column compare — joins, table-aliased columns, value remapping, "all" sentinels — define `filter{StudlyField}(mixed $value, string $operator): void` and drive `$this->query` yourself. The field must still be in `$filterable`; the custom method takes precedence over the standard apply. (`$value` is typed `mixed` because the parsed filter value can be a scalar or an array depending on operator — narrow it inside the method; only use a tighter type hint if your allowed operators guarantee that shape.)

```php
public function filterCategory(mixed $value, string $operator): void
{
    if ($value === 'all') {
        return;                                  // short-circuit → no constraint
    }
    if (is_array($value)) {
        $this->query->whereIn('a.category', $value);          // note the 'a.' table alias
    } elseif ($operator === 'ne') {
        $this->query->where('a.category', '!=', $value);
    } else {
        $this->query->where('a.category', '=', $value);
    }
}
```

- Use the **table alias/prefix** (`a.category`) inside custom methods when the base query joins or aliases tables, so the column is unambiguous.
- Use the explicit 3-arg `where('col', '=', $v)` form (the 2-arg shorthand treats a string value as the operator).

## Safe public filtering

- **Don't leave `$filterable`/`$sortable` as `null` on public endpoints** — `null` means *every* field is filterable/sortable, which leaks queryability over internal columns. Explicitly list the allowed fields.
- **Clamp sensitive values in a custom method.** E.g. only ever expose active rows publicly:
  ```php
  public function filterStatus(string $value, string $operator): void
  {
      if ($value !== 'published') {
          return;                  // ignore attempts to filter to other statuses publicly
      }
      $this->query->where('a.status', '=', 'published');
  }
  ```
- Validate enum-like values in the custom method (whitelist valid `category` values, etc.) rather than passing user input straight through.

## Two filtering surfaces (don't confuse them)

- **`Glueful\Api\Filtering\QueryFilter`** — the request-driven filter class above (the one you almost always want for list endpoints).
- **`Glueful\Repository\Concerns\QueryFilterTrait`** — a repository-side helper for programmatic filtering inside a repository method; not request-parsing. Use the `QueryFilter` class for endpoint filtering.

## Gotchas

- **`null` filterable = unrestricted**, not "none" — the opposite of safe. Always list fields for public endpoints.
- **Custom method name is `filter` + StudlyCase field** (`filterCategory` for `category`); a mismatched name silently falls back to the standard filter.
- **Apply in the service**, not the controller — keeps HTTP and orchestration separate (`glueful-architect-app`).
- **Table-prefix columns** in custom methods when the query uses aliases/joins.

## Checklist

- [ ] Filter extends `Glueful\Api\Filtering\QueryFilter`; created via `scaffold:filter` or by hand.
- [ ] `$filterable` and `$sortable` are explicit allow-lists (not `null`) on any public endpoint; `$searchable` set; `$defaultSort` chosen.
- [ ] Custom `filter{Field}()` methods use the table alias, the 3-arg `where()`, and clamp/validate sensitive or enum values.
- [ ] Applied in the service via `(new XFilter($request))->apply($query)->paginate(...)`.
