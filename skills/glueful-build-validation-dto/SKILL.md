---
name: glueful-build-validation-dto
description: Validate and normalize request input in Glueful — a typed RequestData DTO with Rule-attribute constraints (router-hydrated, auto-422) by default, or a manual DTO whose static factory runs the Validator (Sanitize/Length/InArray/Numeric/Required/DbUnique/… rule objects), throws ValidationException on failure, and builds the typed object from the sanitized values. Also covers custom Rule classes and RuleRegistry. Use when accepting input for a write/list endpoint. Glueful validation is Validator + Rule objects and rule strings, not Laravel's Validator facade or $request->validate().
---

# Validation DTOs

A DTO validates and normalizes request input **at the edge of a use case**, so controllers stay thin and validation doesn't drift across endpoints (`glueful-architect-app`). A failed validation throws `Glueful\Validation\ValidationException`, rendered as **422** with a top-level `errors` map (`field => [messages]`).

## Pick a style

1. **Typed `RequestData` DTO (default).** A class implementing `Glueful\Validation\Contracts\RequestData` with `#[Rule('required|string|max:200')]` on its promoted constructor params. The **router** decodes the JSON body, validates, and injects it typed — no factory call — auto-`422` on failure; the OpenAPI generator derives the request schema from it. Scaffold: `php glueful scaffold:dto CreateArticleData` (→ `app/DTOs`).
2. **Manual `Validator` DTO (this skill's main example).** A `final` class + static factory that runs the `Validator` with Rule objects and builds from `filtered()`. Use it when you need **mutating sanitization** (`Sanitize`/`CastTo*` — `RequestData` validates but never transforms), non-JSON input (form/multipart), or validation outside HTTP (services, jobs, imports).
3. **`#[Validate([...])]` / Glueful's own `FormRequest`** — rule strings run by the `validate` route middleware (`scaffold:request` → `app/Http/Requests`). Fine for array-style input you haven't migrated; don't combine `#[Validate]` with a `RequestData` param on one route.

## Typed `RequestData` DTO

```php
namespace App\DTOs;

use Glueful\Validation\Attributes\{ArrayOf, FromRoute, FromQuery, Rule};
use Glueful\Validation\Contracts\{RequestData, ValidatesSelf};

final class UpdateArticleData implements RequestData, ValidatesSelf
{
    public function __construct(
        #[FromRoute] public readonly string $uuid,                       // from {uuid} in the path
        #[Rule('required|string|max:200')] public readonly string $title,
        #[Rule('in:draft,published')] public readonly string $status = 'draft',
        #[Rule('array'), ArrayOf('string')] public readonly array $tags = [],
        #[Rule('array'), ArrayOf(LinkData::class)] public readonly array $links = [],  // nested RequestData
        #[FromQuery] public readonly bool $notify = false,             // ?notify=true → coerced to bool
        #[Rule('nullable|date')] public readonly ?string $publish_at = null,
    ) {}

    public function validate(): array                      // cross-field; runs after construction
    {
        return ($this->status === 'published' && $this->publish_at === null)
            ? ['publish_at' => ['Required when publishing.']]
            : [];
    }
}
```

- **Exact-name binding** — the param name IS the JSON/path/query key (`$publish_at` ↔ `publish_at`); no snake/camel conversion.
- A required-but-absent non-nullable param is a **422**, not a `TypeError`; defaults/nullables fill absent keys; scalars are coerced to the declared type after validation.
- `#[ArrayOf]` errors are dot-pathed (`links.0.url`); nesting is capped at 5 levels. One source attribute per field; `#[FromRoute]` needs a matching `{placeholder}` (misuse fails loud).
- Path/query values arrive as **strings**, and the `integer`/`boolean` rules are strict type checks. Since 1.91.3 a route/query string is converted to the field's declared `int`/`float`/`bool` before the rules run (`true/false/1/0/yes/no/on/off` for booleans), and one that doesn't convert is a 422. Before 1.91.3, use `numeric` (or no type rule) on those fields.
- JSON body only — a form/multipart request won't hydrate; use style 2 or 3 there.

Rule strings (`#[Rule]`, `#[Validate]`, `FormRequest`): `required`, `string`, `integer`/`int`, `boolean`/`bool`, `array`, `numeric`, `min:N`/`max:N` (a string's length, a number's value or an array's count since 1.91.3; string length only before), `in:a,b`, `regex:/…/`, `email`, `url`, `json`, `uuid`, `date[:fmt]`, `before:`/`after:`, `confirmed`, `unique:table,column[,exceptId]`, `exists:table,column`, `nullable`, `sometimes`, `file:…`, `image:…`, `dimensions:…`. Every rule except `required` passes on an absent/`null` value. `nullable` also treats `""` as no value; `sometimes` skips an absent key entirely (partial updates). `unique:`/`exists:` really query the DB (since 1.91) — still back uniqueness with a unique index.

## Manual `Validator` DTO

```php
namespace App\DTOs\Article;

use Glueful\Bootstrap\ApplicationContext;
use Glueful\Validation\Validator;
use Glueful\Validation\ValidationException;
use Glueful\Validation\Rules\{Sanitize, Required, Length, InArray};

final class ArticleCreateDTO
{
    public function __construct(
        public readonly string  $title,
        public readonly ?string $body,
        public readonly ?string $category,
        public readonly string  $status,
        /** @var array<string> */
        public readonly array   $tagUuids = [],
    ) {}

    /**
     * @param array<string,mixed> $data Raw request data
     * @throws ValidationException
     */
    public static function fromRequest(array $data, ApplicationContext $context): self
    {
        $validator = new Validator([
            'title'    => [new Sanitize(['trim']), new Required(), new Length(1, 200)],
            'body'     => [new Sanitize(['trim']), new Length(0, 50000)],
            'category' => [new Sanitize(['trim']), new InArray(['news', 'opinion', 'review'])],
            'status'   => [new Sanitize(['trim']), new InArray(['draft', 'published'])],
        ], $context);                                // context lets DbUnique/Exists reach the DB

        $errors = $validator->validate($data);      // returns an errors map; does NOT throw
        if ($errors !== []) {
            throw new ValidationException($errors);
        }

        $clean = $validator->filtered();             // sanitized values — use THESE, not raw $data

        return new self(
            title:    $clean['title'],
            body:     ($clean['body'] ?? '') !== '' ? $clean['body'] : null,
            category: $clean['category'] ?? null,
            status:   $clean['status'] ?? 'draft',
            tagUuids: array_values(array_filter((array) ($data['tags'] ?? []), 'is_string')),
        );
    }
}
```

### How the Validator works

- `new Validator(array $rules, ?ApplicationContext $context = null, ?Connection $connection = null)` — a **field → array-of-Rule-objects** map. `Glueful\Validation\Support\Rules::of([...])` is shorthand (no context).
- `validate(array $data): array` returns an **errors map**, empty when valid — it **does not throw**.
- Rules run in order per field. **Mutating rules** (`Sanitize`, `CastToInt`, `CastToBoolean`, `CastToDate`) transform the value for the rules after them — put them first. `filtered()` returns the post-mutation values for **ruled fields only** (unruled keys are dropped; `sometimes`-skipped fields are omitted).
- Built-in rule objects (`Glueful\Validation\Rules\*`): `Required`, `Nullable`, `Sometimes`, `Email`, `Url`, `Uuid`, `Json`, `Regex`, `Type('integer')` (strict), `Length(min,max)`, `Range(min,max)`, `Numeric(min,max,integerOnly)`, `InArray([...])`, `Date`, `Before`/`After`, `Confirmed`, `DbUnique`, `Exists`, `File`, `Image`, `Dimensions`, `Sanitize([...])` (`trim`, `ltrim`, `rtrim`, `strip_tags`, `lower`, `upper`), `CastTo*`.
- **`DbUnique`/`Exists` fail closed**: a `Validator` built without context or connection throws `LogicException` when they run on a non-empty value. Pass `$context` (or `connection: db($context)`).

### Use it in a controller

```php
public function store(Request $request): Response
{
    $dto = ArticleCreateDTO::fromRequest(RequestHelper::getRequestData($request), $this->getContext());
    $article = $this->articleService->create($dto, $this->userContext->getUserUuid());
    return $this->created($article, 'Article created');
}
```

`ValidationException` is mapped to the 422 envelope by the exception handler — don't catch it in the controller. For one-off checks, `ValidationException::forField($field, $msg)` / `forFields([...])` exist.

## Custom rules

Implement `Glueful\Validation\Contracts\Rule` — return an error message string, or `null` when valid. Let empty/null pass (so `Required` owns presence). Scaffold with `php glueful scaffold:rule ReservedUsername` (→ `app/Validation/Rules`).

```php
namespace App\Validation\Rules;

use Glueful\Validation\Contracts\Rule;

final class ReservedUsername implements Rule
{
    public function __construct(private readonly ?string $message = null) {}

    public function validate(mixed $value, array $context = []): ?string
    {
        if ($value === null || $value === '') {
            return null;                          // presence is Required's job
        }
        if (!is_string($value)) {
            return 'Expected string.';
        }
        return \App\Support\ReservedUsernames::isReserved($value)
            ? ($this->message ?? 'This username is reserved.')
            : null;
    }
}
```

- As an object: `'username' => [new Required(), new Length(3, 30), new ReservedUsername()]`. `$context` carries `['field' => ..., 'data' => $allData]` (plus `'context'` when the Validator has one) for cross-field rules.
- By name in rule strings (`#[Rule('required|reserved_username')]`): an `App\Validation\Rules\{StudlyName}` class is resolved automatically; otherwise register it in a provider's `boot()` — `app($context, RuleRegistry::class)->register('reserved_username', ReservedUsername::class)` (`Glueful\Validation\Support\RuleRegistry`). Built-in names can't be shadowed.
- For a value *transform*, implement `MutatingRule::mutate(mixed $value, array $context = []): mixed` instead.

## Gotchas

- **`validate()` returns errors, it doesn't throw** — throw `ValidationException($errors)` when the map is non-empty.
- **Build the manual DTO from `filtered()`**, not raw `$data`, or you lose the `Sanitize` mutations.
- **Absent ≠ invalid**: without `Required`/`required`, a missing field passes every rule.
- **DB rules need a connection** — give the `Validator` the context, or they throw.
- **`RequestData` never mutates input** and reads JSON only — trimming/casting or form input → manual DTO.
- **Validate in the DTO, not the controller** — the controller just calls the factory or type-hints the `RequestData`.

## Checklist

- [ ] Chose the style: typed `RequestData` by default; manual `Validator` DTO for sanitization, non-JSON input, or non-HTTP use.
- [ ] `RequestData`: params named exactly as the JSON keys; `#[Rule]` on each; `#[ArrayOf]` for arrays; `#[FromRoute]`/`#[FromQuery]` for non-body fields; `ValidatesSelf` for cross-field rules.
- [ ] Manual DTO: `final` class, readonly constructor, static factory building `new Validator([...], $context)`, throwing `ValidationException($errors)`, constructing from `filtered()`.
- [ ] Mandatory fields carry `Required`/`required`; partial updates use `sometimes`.
- [ ] Custom rules implement `Rule::validate(): ?string` (null = pass) in `App\Validation\Rules`, or `MutatingRule` for transforms.
- [ ] Controller passes the typed DTO to a service; no inline validation.
