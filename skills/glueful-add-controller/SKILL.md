---
name: glueful-add-controller
description: Add or modify an HTTP controller in a Glueful framework project — extending BaseController, registering it in the container, reading and validating request input (typed RequestData DTOs or manual), resolving the authenticated user, and returning the framework Response envelope or a typed ResponseData. Use when building API endpoints in a Glueful app. Glueful controllers are NOT Laravel controllers — there is no request()/response() helper, no abort(), no $this->db, and the response envelope is Glueful\Http\Response, not Illuminate.
---

# Adding a Glueful Controller

Glueful controllers extend `Glueful\Controllers\BaseController` and return `Glueful\Http\Response` (which extends Symfony's `JsonResponse`). There is no `request()` global, no `response()->json()`, no `abort()`. This is a JSON API framework. Scaffold one with `php glueful scaffold:controller WidgetController [--resource|--api]` (writes `app/Controllers`, namespace `App\Controllers`, and prints the registration step — it does not register it or add a route).

## Skeleton

```php
<?php

declare(strict_types=1);

namespace App\Controllers;

use Glueful\Bootstrap\ApplicationContext;
use Glueful\Controllers\BaseController;
use Glueful\Helpers\RequestHelper;
use Glueful\Http\Response;
use Glueful\Validation\ValidationException;
use Symfony\Component\HttpFoundation\Request;

final class WidgetController extends BaseController
{
    // BaseController::__construct requires ApplicationContext FIRST. Extra typed
    // params are autowired by the container (they must themselves be registered/resolvable).
    public function __construct(
        ApplicationContext $context,
        private readonly WidgetService $widgets,
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

## Register it in the container (required)

The router resolves `[WidgetController::class, 'method']` from the container with **no autowire fallback** — an unregistered controller fails with "Service … not found" on its first hit. In the skeleton, add it to `app/Providers/AppServiceProvider::services()`:

```php
\App\Controllers\WidgetController::class => ['autowire' => true, 'shared' => true],
\App\Services\WidgetService::class       => ['autowire' => true, 'shared' => true],   // its deps too
```

(Extensions register controllers in their provider's `services()` the same way.) Never `new` a controller yourself.

## What BaseController gives you

- `$this->getContext()` / `$this->context` — the `ApplicationContext` for helpers: `db($this->getContext())`, `config(...)`, `app(...)`.
- `$this->currentUser` (`?UserIdentity`), `$this->userContext`, `$this->request`, `$this->authManager`, `$this->repositoryFactory`.
- **There is no `$this->db`** — use `db($this->getContext())->table('widgets')` or inject `Glueful\Database\Connection`. `insert()` returns the affected-row count; use `insertGetId()` for the new id.

## Reading request input

`RequestHelper::getRequestData($request): array` (or `$this->getRequestData()`) normalizes JSON / form / query input. Handler params are resolved by type then name: the Symfony `Request`, a typed `RequestData` DTO, container services, and route params by name (`{id}` → `int $id`, cast to the declared scalar type) — order doesn't matter.

## Validating input

Prefer a typed **`RequestData`** parameter (below) — the router hydrates + validates it and auto-`422`s. Otherwise throw `Glueful\Validation\ValidationException` (rendered as **422** with a top-level `errors` map):

```php
throw ValidationException::forField('email', 'Email address is required');
throw ValidationException::forFields(['email' => '...', 'password' => '...']);
```

…or call a manual `Validator` DTO (`SomeDTO::fromRequest($data)`; see `glueful-build-validation-dto`). `$this->validateRequest($data, $rules)` is a legacy helper that only knows `required`/`email`/`max:N` — don't use it for new code.

## Resolving the authenticated user & authorization

Use the request user context — **not** `$request->getUser()` (Symfony's accessor returns the HTTP Basic Auth username):

```php
$userUuid = $this->userContext->getUserUuid();   // ?string
if ($userUuid === null) {
    throw new \Glueful\Http\Exceptions\Domain\AuthenticationException('Authentication required');
}
$user = $this->userContext->getUser();            // ?Glueful\Auth\UserIdentity
```

`UserIdentity` is immutable with **method** accessors: `uuid()`, `email()`, `username()`, `roles()`, `scopes()`, plus a claims bag. It's also on `$this->currentUser`.

Authorization: declaratively with `#[RequiresPermission('widgets.edit')]` / `#[RequiresRole('editor')]` (`Glueful\Auth\Attributes`, on the method or class; `gate_permissions` is auto-attached; stacked roles = any-of, stacked permissions = all-of), or imperatively with `$this->requirePermission('widgets.edit', 'widgets')` (throws) / `$this->can(...)` (bool). Permission checks need a permission provider (e.g. `glueful/aegis`).

## Returning responses

Two equivalent surfaces — both return `Glueful\Http\Response`:

- **`BaseController` helpers**: `$this->success($data, $msg)`, `created(...)`, `notFound($msg)`, `unauthorized($msg)`, `forbidden($msg)`, `validationError($errors)`, `serverError($msg)`, `paginated($data, $meta)`, plus HTTP-cache helpers (`cached()`, `checkNotModified()`).
- **`Response` statics** (anywhere): `Response::success($data, $msg)`, `created(...)`, `noContent()` (204), `error($msg, $status)`, `validation($errors)`, `notFound(...)`, `paginated($items, $total, $page, $perPage)`.

The envelope is `{success, message, data}` (errors: `{success: false, message, error: {...}}`). Don't return raw arrays or build a Symfony `Response`/`JsonResponse` by hand. Prefer throwing `Glueful\Http\Exceptions\Client\*` / `Domain\*` exceptions (e.g. `NotFoundException`, `ConflictException`, `BusinessLogicException`) from services — the handler maps them to status codes.

**Type your action return** as `Response` (or a `ResponseData` type, below). For a bodiless 204 use `Response::noContent()`.

## Typed request/response DTOs (auto-documented)

An action can take a **typed `RequestData` parameter** and/or return a **typed `ResponseData`** — the router hydrates, validates, and envelopes them, and the reflect OpenAPI generator derives the schemas from the types. Prefer this for fixed, statically-typed JSON payloads. Scaffold with `php glueful scaffold:dto CreateWidgetData` / `scaffold:dto WidgetData --response` (writes `app/DTOs`).

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

- **`RequestData` param** — decoded from the **JSON body** (no form fallback; invalid JSON → 422), validated against `#[Rule]` strings, injected typed; failure auto-returns the standard `422`. Keys bind to constructor params **by exact name** (no snake/camel conversion). Arrays/nested DTOs via `#[ArrayOf('int')]` / `#[ArrayOf(ItemData::class)]` (dot-path 422s, never a 500); path/query fields via `#[FromRoute]` / `#[FromQuery]`; cross-field checks via `ValidatesSelf::validate()`; custom rules by name. Details in `glueful-build-validation-dto`.
- **`ResponseData` return** — serialized and wrapped in the envelope; `#[ResponseStatus(2xx)]` sets the status; implement `Glueful\Http\Contracts\HasResponseMessage::responseMessage()` for a custom `message`. Lists: `new CollectionResponse($items)` / `new PaginatedResponse($items, $page, $perPage, $total)` (`Glueful\Http\Responses`) — annotate `@return CollectionResponse<WidgetData>` so OpenAPI knows the item type. A returned `JsonResource`/`ResourceCollection` is normalized through its own `toResponse()`.
- **Stay manual** for polymorphic bodies, multipart/form input, binary/stream responses, or responses that set their own headers (caching/ETag). Document those with `#[ApiResponse]`/`#[ApiRequestBody]` (see `glueful-add-route`).

Details: framework `docs/REQUEST_DTOS.md`, `docs/RESPONSE_DTOS.md`.

## Wiring the controller to a route

Either register in `routes/*.php` (app routes aren't auto-prefixed — group them):

```php
$router->get('/v1/widgets/{id}', [WidgetController::class, 'show'])->where('id', '\d+');
```

…or use attribute routing on the controller (scanned from `app/Controllers` at boot; still needs container registration; put the full prefix in `#[Controller]`):

```php
use Glueful\Routing\Attributes\{Controller, Get};

#[Controller(prefix: '/v1/widgets')]
final class WidgetController extends BaseController
{
    #[Get('/{id}', where: ['id' => '\d+'])]
    public function show(int $id): Response { /* ... */ }
}
```

(See `glueful-add-route` for middleware, rate limiting, scopes, and OpenAPI attributes.)

## Not-Laravel reminders

- No `request()` / `response()` globals → action `Request` param + `RequestHelper::getRequestData()`; return `Response::success()`.
- No `abort(404)` → `return $this->notFound(...)` or throw the matching `Glueful\Http\Exceptions\*` exception.
- No `$request->user()` / `auth()->user()` → `$this->userContext->getUser()` / `$this->currentUser`.
- No `$this->db` → `db($this->getContext())` or an injected `Connection`.
- No `Validator::make()` / `$request->validate()` → typed `RequestData`, `ValidationException::forField(...)`, or a DTO.
- Controllers are container-resolved and must be registered → declare dependencies as constructor params (after `ApplicationContext`).

## Checklist

- [ ] Extends `BaseController`; constructor calls `parent::__construct($context)` with `ApplicationContext` first.
- [ ] Controller (and its injected services) registered in `AppServiceProvider::services()` with `'autowire' => true`.
- [ ] Input via a typed `RequestData` param, or `RequestHelper::getRequestData($request)` + `ValidationException`/DTO.
- [ ] Authenticated user via `$this->userContext` / `$this->currentUser`, not `$request->getUser()`; authz via `#[RequiresPermission]`/`#[RequiresRole]` or `requirePermission()`.
- [ ] Returns the `Response` envelope (typed `: Response`) or a typed `ResponseData`/`CollectionResponse`/`PaginatedResponse`.
- [ ] Wired to a route (route file or `#[Controller]`/`#[Get]`… attributes).
