---
name: glueful-build-validation-dto
description: Validate and normalize request input with a Glueful DTO — a final class with a readonly constructor and a static factory that runs the Validator (Sanitize/Length/InArray/Numeric/Required/… rules), throws ValidationException on failure, and builds the typed object from the sanitized values. Also covers writing custom Rule classes. Use when accepting input for a write/list endpoint. Glueful validation is Validator + Rule objects, not Laravel's Validator facade or FormRequest.
---

# Validation DTOs

A DTO validates and normalizes request input **at the edge of a use case**, so controllers stay thin and validation doesn't drift across endpoints (`glueful-architect-app`). It's a `final` class with a readonly constructor (the validated fields) and a static factory that runs the framework `Validator`, throws `ValidationException` on failure, and constructs from the **sanitized** values.

## Define the DTO

```php
namespace App\DTO\Article;

use Glueful\Validation\Validator;
use Glueful\Validation\ValidationException;
use Glueful\Validation\Rules\{Sanitize, Length, InArray};

final class ArticleCreateDTO
{
    public function __construct(
        public readonly ?string $title,
        public readonly ?string $body,
        public readonly ?string $category,
        public readonly ?string $status,
        /** @var array<string> */
        public readonly array   $tagUuids = [],
    ) {}

    /**
     * @param array<string,mixed> $data Raw request data
     * @throws ValidationException
     */
    public static function fromRequest(array $data): self
    {
        $validator = new Validator([
            'title'    => [new Sanitize(['trim']), new Length(1, 200)],
            'body'     => [new Sanitize(['trim']), new Length(0, 50000)],
            'category' => [new Sanitize(['trim']), new InArray(['news', 'opinion', 'review'])],
            'status'   => [new Sanitize(['trim']), new InArray(['draft', 'published'])],
        ]);

        $errors = $validator->validate($data);   // returns an errors map; does NOT throw
        if (!empty($errors)) {
            throw new ValidationException($errors);
        }

        $clean = $validator->filtered();          // sanitized/mutated values — use THESE, not raw $data

        return new self(
            title:    $clean['title'] ?? null,
            body:     ($clean['body'] ?? '') !== '' ? $clean['body'] : null,
            category: $clean['category'] ?? null,
            status:   $clean['status'] ?? 'draft',
            tagUuids: array_values(array_filter((array)($data['tags'] ?? []), 'is_string')),
        );
    }
}
```

(The framework's own DTOs name the factory `from()`; `fromRequest()` is a clearer convention — the name is yours, the mechanism is what matters.)

## How the Validator works

- Construct with a **field → array-of-Rule-objects** map: `new Validator(['email' => [new Required(), new Email()]])`.
- `validate(array $data): array` returns an **errors map** (`field => [messages]`), empty when valid — it **does not throw**; you decide to throw `ValidationException($errors)`.
- `filtered()` returns the values **after mutating rules ran** (e.g. `Sanitize` trimmed/stripped). Always build the DTO from `filtered()`, not the raw input, so you persist the cleaned values.

Built-in rules live in `Glueful\Validation\Rules\*`: `Required`, `Email`, `Length(min,max)`, `InArray([...])`, `Numeric(min,max)`, `Before`/`After`, and `Sanitize([...])`. `Sanitize` is a **mutating** rule (transforms the value), not a check — that's why it shows up in `filtered()`.

## Use it in a controller

The controller validates via the DTO and hands the **typed object** to a service — it does not hand-roll checks:

```php
public function store(Request $request): Response
{
    $dto = ArticleCreateDTO::fromRequest(RequestHelper::getRequestData($request)); // throws ValidationException on bad input
    $article = $this->articleService->create($dto, $this->userContext->getUserUuid());
    return $this->created($article, 'Article created');
}
```

`ValidationException` is mapped to a structured 4xx by the framework's exception handler — you don't catch it in the controller. (For one-off checks outside a DTO, `ValidationException::forField($field, $msg)` / `forFields([...])` exist.)

## Custom rules

Implement `Glueful\Validation\Contracts\Rule` — return an error message string, or `null` when valid. Let empty/null pass (so `Required` owns presence):

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

Use it like any built-in: `'username' => [new Required(), new Length(3, 30), new ReservedUsername()]`. The `$context` arg carries `['field' => ..., 'data' => $allData]` for cross-field rules. For a value *transform* (not a check), implement `MutatingRule` instead and return the mutated value from `mutate()`.

## Gotchas

- **`validate()` returns errors, it doesn't throw** — you throw `ValidationException($errors)` when the map is non-empty.
- **Build the DTO from `filtered()`**, not raw `$data`, or you lose the `Sanitize` mutations (untrimmed/unstripped values get persisted).
- **`Sanitize` is a mutating rule**, not a guard — it never produces an error, it changes the value.
- **Custom rules return `?string`** (message or `null`); let null/empty pass and leave presence to `Required`.
- **Validate in the DTO, not the controller** — prevents per-endpoint validation drift; the controller just calls `XDTO::fromRequest(...)`.

## Checklist

- [ ] DTO is a `final` class with a readonly constructor of the validated fields + a static `fromRequest()`/`from()` factory.
- [ ] Factory builds a `Validator([field => [Rule...]])`, throws `ValidationException($errors)` when `validate()` returns a non-empty map.
- [ ] DTO constructed from `$validator->filtered()` (sanitized values), not raw input.
- [ ] Custom rules implement `Rule::validate(): ?string` (null = pass), or `MutatingRule` for transforms.
- [ ] Controller calls the DTO factory and passes the typed DTO to a service; no inline validation.
