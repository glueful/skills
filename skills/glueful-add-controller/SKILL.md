---
name: glueful-add-controller
description: Add or modify an HTTP controller in a Glueful framework project — extending BaseController, reading and validating request input, resolving the authenticated user, and returning the framework Response envelope. Use when building API endpoints in a Glueful app. Glueful controllers are NOT Laravel controllers — there is no request()/response() helper, no abort(), and the response envelope is Glueful\Http\Response, not Illuminate.
---

# Adding a Glueful Controller

Glueful controllers extend `Glueful\Controllers\BaseController` and return `Glueful\Http\Response` (which extends Symfony's `JsonResponse`). There is no `request()` global, no `response()->json()`, no `abort()`. This is a JSON API framework.

## Skeleton

```php
<?php

namespace App\Controllers;   // or wherever the app's controllers live

use Glueful\Bootstrap\ApplicationContext;
use Glueful\Controllers\BaseController;
use Glueful\Helpers\RequestHelper;
use Glueful\Http\Response;
use Glueful\Validation\ValidationException;
use Symfony\Component\HttpFoundation\Request;

final class WidgetController extends BaseController
{
    // BaseController::__construct requires ApplicationContext FIRST. Extra typed
    // constructor params are auto-wired by the container, which resolves the
    // controller (do NOT new it up manually).
    public function __construct(
        ApplicationContext $context,
        private WidgetService $widgets,
    ) {
        parent::__construct($context);
    }

    public function index(Request $request): Response
    {
        $result = $this->widgets->paginate($this->getContext());
        return $this->success($result, 'Widgets retrieved');
    }

    public function show(Request $request, int $id): Response
    {
        $widget = $this->widgets->find($this->getContext(), $id);
        if ($widget === null) {
            return $this->notFound('Widget not found');
        }
        return $this->success($widget);
    }

    public function store(Request $request): Response
    {
        $data = RequestHelper::getRequestData($request);   // NOT request()->all()
        if (!isset($data['name']) || $data['name'] === '') {
            throw ValidationException::forField('name', 'Name is required');
        }
        $widget = $this->widgets->create($this->getContext(), $data);
        return $this->created($widget, 'Widget created');
    }
}
```

## Reading request input

Use `RequestHelper::getRequestData($request): array` — it normalizes JSON / form / query input. The action receives the Symfony `Request` as a parameter (the router passes it); route path params (`{id}`) arrive as typed method arguments after the request.

## Validating input

Throw `Glueful\Validation\ValidationException` (the exception handler maps it to a 4xx envelope):

```php
throw ValidationException::forField('email', 'Email address is required');
throw ValidationException::forFields(['email' => '...', 'password' => '...']);
```

Or validate through a DTO — either a manual `Validator` DTO (`SomeDTO::from($data)` throws `ValidationException`; see `glueful-build-validation-dto`) or, for a fixed JSON body, a typed **`RequestData`** parameter that the router hydrates + validates for you and auto-`422`s on failure (see *Typed request/response DTOs* below).

## Resolving the authenticated user

Use the request user context — **not** `$request->getUser()` (Symfony's accessor returns the HTTP Basic Auth username):

```php
$userUuid = $this->userContext->getUserUuid();   // ?string
if ($userUuid === null) {
    throw new \Glueful\Http\Exceptions\Domain\AuthenticationException('Authentication required');
}
$user = $this->userContext->getUser();            // ?Glueful\Auth\UserIdentity ($user->uuid(), $user->email())
```

> As of framework 1.50.0 the user is a canonical, immutable `Glueful\Auth\UserIdentity` (accessors are **methods**: `$user->uuid()`, `$user->email()`, plus `roles()`/`scopes()` and an open claims bag) — the old `AuthenticatedUser` class was removed when the concrete user store moved to the `glueful/users` extension. `BaseController` also exposes it as `$this->currentUser` (already initialized in the constructor).

## Returning responses

Two equivalent surfaces — both return `Glueful\Http\Response`:

- **`BaseController` helpers** (inside a controller): `$this->success($data, $msg)`, `$this->created(...)`, `$this->notFound($msg)`, `$this->unauthorized($msg)`, `$this->forbidden($msg)`, `$this->validationError($errors)`, `$this->serverError($msg)`, `$this->paginated(...)`.
- **`Response` statics** (anywhere): `Response::success($data, $msg)`, `Response::created(...)`, `Response::error(...)`, `Response::notFound(...)`.

The envelope is `{success, message, data}`. Don't return raw arrays or build a `Symfony\…\Response`/`JsonResponse` by hand.

**Type your action return as `Response`.** The project runs `phpstan-strict-rules`, which requires explicit return types on new code. If an action returns heterogeneous response subtypes, the common base is `Glueful\Http\Response` (it extends Symfony's response). Avoid reaching for raw `Symfony\…\Response` just to emit a bodiless 204 — prefer the framework envelope for API consistency; if you genuinely need a 204, `Response::success(null, '…')->setStatusCode(204)` keeps the framework type.

## Typed request/response DTOs (auto-documented; framework ≥ 1.57.0)

Beside the manual `Request` / `RequestHelper::getRequestData()` / `Response::success()` pattern above, an action can take a **typed `RequestData` parameter** and/or return a **typed `ResponseData`** — the router hydrates, validates, and envelopes them for you, and the reflect OpenAPI generator derives the request/response schema straight from the types (no annotation). Prefer this for fixed, flat, statically-typed payloads.

```php
use Glueful\Validation\Contracts\RequestData;
use Glueful\Validation\Attributes\Rule;
use Glueful\Http\Contracts\ResponseData;
use Glueful\Routing\Attributes\ResponseStatus;

final class CreateWidgetData implements RequestData
{
    public function __construct(
        #[Rule('required|string|max:120')] public readonly string $name,
        #[Rule('string|in:active,archived')] public readonly string $status = 'active',
    ) {}
}

final class WidgetData implements ResponseData
{
    public function __construct(
        public readonly string $uuid,
        public readonly string $name,
    ) {}
}

#[ResponseStatus(201)]
public function store(CreateWidgetData $input): WidgetData    // no Request param, no Response::created()
{
    $w = $this->widgets->create($this->getContext(), ['name' => $input->name, 'status' => $input->status]);
    return new WidgetData($w['uuid'], $w['name']);            // auto-enveloped → {success, message, data}
}
```

- **`RequestData` param** — decoded from the request, validated against its `#[Rule]` constraints, injected typed; a failure auto-returns the standard `422`. JSON keys bind to constructor params **by exact name** (no snake/camel conversion). **As of v2 (framework ≥ 1.58.0)** it is no longer flat-scalar-only: `array`/nested-DTO fields hydrate via `#[ArrayOf('int')]` / `#[ArrayOf(ItemData::class)]` (recursive, dot-path `422`s, never a `TypeError`/500); fields can come from the path/query via `#[FromRoute]`/`#[FromQuery]` (body is the default; one source per field); add a `ValidatesSelf::validate()` cross-field hook; and use app-registered custom rules (`RuleRegistry`) by name in `#[Rule]`. Misuse (a field with both source attributes, a source attribute on a nested DTO, or a `#[FromRoute]` with no matching `{placeholder}`) fails loud.
- **`ResponseData` return** — serialized and wrapped in the `{success, message, data}` envelope automatically; `#[ResponseStatus(2xx)]` sets the status; implement `Glueful\Http\Contracts\HasResponseMessage` to supply a custom envelope `message`. Lists use `Glueful\Http\Responses\CollectionResponse` / `PaginatedResponse`. A returned `JsonResource`/`ResourceCollection` is auto-normalized through its own `toResponse()`.
- **Stay manual** (the pattern at the top of this skill) for: polymorphic/varying bodies, multipart/base64 input, binary/stream responses, or responses that set their own headers (caching/ETag). Document those with `#[ApiResponse]`/`#[ApiRequestBody]` (see `glueful-add-route`).

Details: framework `docs/REQUEST_DTOS.md`, `docs/RESPONSE_DTOS.md`; for the manual-validation `Validator` DTO see `glueful-build-validation-dto`.

## Wiring the controller to a route

Either register in `routes/*.php`:

```php
$router->get('/widgets/{id}', [WidgetController::class, 'show'])->where('id', '\d+');
```

…or use attribute routing on the controller:

```php
use Glueful\Routing\Attributes\{Controller, Get, Post};

#[Controller(prefix: '/widgets')]
final class WidgetController extends BaseController
{
    #[Get('/{id}', where: ['id' => '\d+'])]
    public function show(Request $request, int $id): Response { /* ... */ }
}
```

(See the `glueful-add-route` skill for middleware, rate limiting, and the route file/manifest details.)

## Not-Laravel reminders

- No `request()` / `response()` globals → action `Request` param + `RequestHelper::getRequestData()`; return `Response::success()`.
- No `abort(404)` → `return $this->notFound(...)` or throw the matching `Glueful\Http\Exceptions\*` exception.
- No `$request->user()` for the app user → `$this->userContext->getUser()`.
- No `Validator::make()` → `ValidationException::forField(...)` or a DTO.
- Controllers are container-resolved → declare dependencies as constructor params (after `ApplicationContext`); don't instantiate controllers manually.

## Checklist

- [ ] Extends `BaseController`; constructor calls `parent::__construct($context)` with `ApplicationContext` first.
- [ ] Dependencies declared as typed constructor params (container auto-wires them).
- [ ] Input read via `RequestHelper::getRequestData($request)`; validated via `ValidationException`/DTO.
- [ ] Authenticated user via `$this->userContext`, not `$request->getUser()`.
- [ ] Returns the `Response` envelope (`$this->success(...)` / `Response::success(...)`); action typed `: Response`. (Or returns a typed `ResponseData`/`CollectionResponse`/`PaginatedResponse` for auto-enveloping + auto-OpenAPI.)
- [ ] Wired to a route (route file or `#[Controller]`/`#[Get]`… attributes).
