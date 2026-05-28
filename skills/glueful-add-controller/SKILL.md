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

Or validate through a DTO (`SomeDTO::from($data)` throws `ValidationException` on failure) — see the framework's `*DTO` classes for the attribute-based rule pattern.

## Resolving the authenticated user

Use the request user context — **not** `$request->getUser()` (Symfony's accessor returns the HTTP Basic Auth username):

```php
$userUuid = $this->userContext->getUserUuid();   // ?string
if ($userUuid === null) {
    throw new \Glueful\Http\Exceptions\Domain\AuthenticationException('Authentication required');
}
$user = $this->userContext->getUser();            // ?AuthenticatedUser ($user->email, etc.)
```

## Returning responses

Two equivalent surfaces — both return `Glueful\Http\Response`:

- **`BaseController` helpers** (inside a controller): `$this->success($data, $msg)`, `$this->created(...)`, `$this->notFound($msg)`, `$this->unauthorized($msg)`, `$this->forbidden($msg)`, `$this->validationError($errors)`, `$this->serverError($msg)`, `$this->paginated(...)`.
- **`Response` statics** (anywhere): `Response::success($data, $msg)`, `Response::created(...)`, `Response::error(...)`, `Response::notFound(...)`.

The envelope is `{success, message, data}`. Don't return raw arrays or build a `Symfony\…\Response`/`JsonResponse` by hand.

**Type your action return as `Response`.** The project runs `phpstan-strict-rules`, which requires explicit return types on new code. If an action returns heterogeneous response subtypes, the common base is `Glueful\Http\Response` (it extends Symfony's response). Avoid reaching for raw `Symfony\…\Response` just to emit a bodiless 204 — prefer the framework envelope for API consistency; if you genuinely need a 204, `Response::success(null, '…')->setStatusCode(204)` keeps the framework type.

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
- [ ] Returns the `Response` envelope (`$this->success(...)` / `Response::success(...)`); action typed `: Response`.
- [ ] Wired to a route (route file or `#[Controller]`/`#[Get]`… attributes).
