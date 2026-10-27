---
name: glueful-add-filter
description: Add filtering, sorting, and full-text search to a Glueful list endpoint with a QueryFilter class — declaring filterable/sortable/searchable fields, a default sort, custom filter{Field}() methods, and safe public filtering. Use when building or hardening a list/index endpoint in a Glueful app. Glueful's QueryFilter is request-driven (?filter[...]/?sort/?search) and applied to a QueryBuilder, not a Laravel scope or Spatie QueryBuilder.
---

# Adding a Query Filter

Glueful list endpoints get filtering/sorting/search from a **`QueryFilter`** subclass (`Glueful\Api\Filtering\QueryFilter`). It parses `?filter[...]`, `?search=`, and `?sort=` from the Symfony `Request` and applies them to a `Glueful\Database\QueryBuilder`. You declare which fields are exposed; everything else is ignored. Applied in the **service** layer (per `glueful-architect-app`), not the controller.

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

    /** Fields included in ?search= (OR across these). */
    protected array $searchable = ['title', 'body'];

    /** Default sort when the request specifies none. '-' prefix = descending; comma-separate several. */
    protected ?string $defaultSort = '-created_at';

    // Optional limits (defaults shown): nesting depth of filter[...] and number of filters.
    protected int $maxDepth = 3;
    protected int $maxFilters = 20;
}
```

Scaffold a starting point with `php glueful scaffold:filter ArticleFilter --filterable=category,status --sortable=created_at --searchable=title,body` (writes `app/Filters`, namespace `App\Filters`).

## Request syntax (what users send)

```
GET /articles?filter[category]=news                          # equality (eq)
GET /articles?filter[category][in]=news,opinion              # IN (comma list or filter[category][in][]=…)
GET /articles?filter[view_count][gte]=100                    # operator
GET /articles?filter[created_at][between]=2026-01-01,2026-06-30
GET /articles?search=climate                                 # OR across $searchable
GET /articles?search=climate&search_fields=title             # narrow search (intersected with $searchable)
GET /articles?sort=-view_count,created_at                    # '-' = desc, comma = multiple
```

Operators (name + aliases): `eq` (`=`), `ne` (`!=`, `neq`), `gt`, `gte`, `lt`, `lte`, `in`, `nin` (`not_in`), `between` (`range`), `contains` (`like`), `starts` (`starts_with`), `ends` (`ends_with`), `null`, `not_null`. `contains`/`starts`/`ends` and `?search=` use the query builder's `whereContains()`-family: case-insensitive on every driver and the term is matched literally (`%`/`_` escaped).

Non-allowed filter/sort fields are **silently ignored** (not applied), so the whitelist is the security boundary.

## Apply it (service layer)

Get the base query from the **repository** (data access stays there), apply the filter in the **service**, then paginate:

```php
// app/Services/Article/ArticleService.php
use App\Filters\ArticleFilter;
use Symfony\Component\HttpFoundation\Request;

public function feed(Request $request, int $page, int $perPage): array
{
    $query = $this->articleRepository->feedQuery();        // repository owns the base query (joins, alias 'a', visibility)
    $query = (new ArticleFilter($request))->apply($query); // service applies request filter/sort/search
    return $query->paginate($page, $perPage);
}
```

`apply(QueryBuilder $query): QueryBuilder` mutates and returns the same builder. The filter reads the request in its constructor — no middleware is required. With an ORM model, filter its underlying builder: `$b = Article::query($context); (new ArticleFilter($request))->apply($b->getQuery()); $b->paginate($page, $perPage);`.

## Custom `filter{Field}()` methods

For anything beyond a plain column compare — joins, table-aliased columns, value remapping, "all" sentinels — define `filter{StudlyField}(mixed $value, string $operator): void` and drive `$this->query` yourself. The field must still be in `$filterable`; the custom method takes precedence over the standard operator. Type `$value` as `mixed` — it is a scalar or an array depending on the request (`filter[x][in][]=…`), and a narrower hint turns a crafted query string into a `TypeError`/500.

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
- Use the explicit 3-arg `where('col', '=', $v)` form (in the 2-arg form a **string** second argument is taken as the operator).
- Underscored/dotted fields studly-case: `author_uuid` → `filterAuthorUuid`.

## Safe public filtering

- **Don't leave `$filterable`/`$sortable` as `null` on public endpoints** — `null` means *every* field is filterable/sortable, which leaks queryability over internal columns. List the allowed fields.
- **Clamp sensitive values in a custom method.** E.g. only ever expose published rows publicly:
  ```php
  public function filterStatus(mixed $value, string $operator): void
  {
      if ($value !== 'published') {
          return;                  // ignore attempts to filter to other statuses publicly
      }
      $this->query->where('a.status', '=', 'published');
  }
  ```
  (Visibility itself belongs in the repository's base query — a filter the client can omit is not an access control.)
- Validate enum-like values in the custom method (whitelist valid `category` values, etc.) rather than passing user input straight through.

## Malformed input

Exceeding `$maxFilters`/`$maxDepth` throws `Glueful\Api\Filtering\Exceptions\InvalidFilterException`; an unknown operator mixed into an operator array, or a `between` without exactly two values, throws `InvalidOperatorException`. Both are plain `InvalidArgumentException`s — not HTTP exceptions — so they render as **500** unless handled. Attach the `filter` middleware to the route (it pre-parses the same params, returns **400** for `InvalidFilterException`, and stores `_filters`/`_sorts`/`_search` request attributes), and/or catch both in the service and rethrow `Glueful\Http\Exceptions\Client\BadRequestException`.

## Two filtering surfaces (don't confuse them)

- **`Glueful\Api\Filtering\QueryFilter`** — the request-driven filter class above (the one you almost always want for list endpoints).
- **`Glueful\Repository\Concerns\QueryFilterTrait`** — a repository-side helper for programmatic filtering inside a repository method; not request-parsing.

## Gotchas

- **`null` filterable = unrestricted**, not "none" — the opposite of safe.
- **Custom method name is `filter` + StudlyCase field**; a mismatched name silently falls back to the standard operator.
- **Apply in the service**, not the controller (`glueful-architect-app`).
- **Table-prefix columns** in custom methods when the query uses aliases/joins.
- **Bad filter syntax is a 500 by default** — add the `filter` middleware or map the exceptions to 400.

## Checklist

- [ ] Filter extends `Glueful\Api\Filtering\QueryFilter`; created via `scaffold:filter` or by hand in `app/Filters`.
- [ ] `$filterable` and `$sortable` are explicit allow-lists (not `null`) on any public endpoint; `$searchable` set; `$defaultSort` chosen.
- [ ] Custom `filter{Field}(mixed $value, string $operator)` methods use the table alias, the 3-arg `where()`, and clamp/validate sensitive or enum values.
- [ ] Applied in the service via `(new XFilter($request))->apply($query)` then `->paginate(...)`.
- [ ] Malformed filters return 400 (`filter` middleware or exception mapping), not 500.
