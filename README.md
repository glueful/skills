# Glueful Skills

Coding-assistant skills for building with the [Glueful](https://github.com/glueful/framework) PHP framework. Installable guides that teach LLMs Glueful's real conventions — `ApplicationContext`, the framework helpers, the ORM and query builder, routing, events, and extensions — and stop them from hallucinating Laravel-style APIs. Compatible with Claude Code, Codex, and similar agent SDKs.

## Why

Glueful is **not Laravel**. The most common failure mode for an AI assistant working in a Glueful codebase is reaching for a Laravel API that doesn't exist (`DB::transaction()`, `Model::create()`, facades, `php artisan`, Blade). These skills encode the framework's actual patterns so assistants write idiomatic Glueful instead of guessing.

## Install

### Universal (Claude Code, Codex, Cursor, and others) — via the `skills` CLI

```bash
# All skills, universal install (writes to .agents/skills)
npx skills add glueful/skills --skill '*' --agent universal --yes

# Or target a specific agent
npx skills add glueful/skills --skill '*' --agent claude-code --yes
npx skills add glueful/skills --skill '*' --agent codex --yes

# Update later
npx skills update
```

### Claude Code — as a plugin

This repo is also a self-contained Claude Code plugin marketplace:

```
/plugin marketplace add glueful/skills
/plugin install glueful-skills@glueful-skills
```

## Skills

| Skill | Use when |
| ----- | -------- |
| [`using-glueful`](skills/using-glueful/SKILL.md) | Writing or modifying any PHP in a Glueful project — controllers, models/ORM, routes, middleware, events, services, extensions, migrations. The baseline "this is not Laravel" conventions. |
| [`glueful-architect-app`](skills/glueful-architect-app/SKILL.md) | Deciding where code belongs in a non-trivial app — the layered map (controllers → DTOs → services → repositories/contracts → models), events/listeners/jobs, filters, app DI wiring, and extension reuse. The composition anchor for the slice skills. |
| [`glueful-write-migration`](skills/glueful-write-migration/SKILL.md) | Adding or changing database schema — `MigrationInterface` + the schema builder, idempotency guards, a working `down()`, and the version-safe `alterTable` form. |
| [`glueful-add-controller`](skills/glueful-add-controller/SKILL.md) | Building API endpoints — `BaseController`, reading/validating input, the authenticated-user context, and the `Response` envelope. |
| [`glueful-add-route`](skills/glueful-add-route/SKILL.md) | Registering & documenting routes — the fluent router, groups, attribute routing, middleware, builder-form rate limiting, and the OpenAPI docblock annotations (`@route`/`@requestBody`/`@response`) that generate the spec/SDK. |
| [`glueful-write-test`](skills/glueful-write-test/SKILL.md) | Writing PHPUnit tests — the SQLite-backed `Connection` harness, mocking framework services, `ApplicationContext`/cache/`JWTService` setup, and the TDD loop. |
| [`glueful-build-orm-query`](skills/glueful-build-orm-query/SKILL.md) | Querying/persisting with the ORM — context-first Model statics, the Builder, eager loading, relations, N+1 safety, and query result caching. |
| [`glueful-add-filter`](skills/glueful-add-filter/SKILL.md) | List-endpoint filtering/sorting/search — a `QueryFilter` subclass (filterable/sortable/searchable allow-lists), custom `filter{Field}()` methods, and safe public filtering. |
| [`glueful-add-event-listener-job`](skills/glueful-add-event-listener-job/SKILL.md) | Side effects — define a `BaseEvent`, dispatch from a service, handle via an `EventSubscriberInterface` subscriber, and defer heavy work to a queued `Job`. |
| [`glueful-build-validation-dto`](skills/glueful-build-validation-dto/SKILL.md) | Validating/normalizing request input — a `final` DTO + `Validator` rule map, `ValidationException`, building from sanitized `filtered()` values, and custom `Rule` classes. |
| [`glueful-create-extension`](skills/glueful-create-extension/SKILL.md) | Packaging an extension — the `glueful-extension` composer manifest, `ServiceProvider`, static `services()` DI, and the `register()`/`boot()` lifecycle. |

Further surfaces (resources, CLI commands, webhooks, uploads, seeders, API versioning) are intentionally deferred until real usage proves they recur — a skill earns its place by *frequency × surprise*, not coverage for its own sake.

Each skill is plain Markdown (`SKILL.md` with `name` + `description` frontmatter), so it works with any agent SDK that reads the skills convention. New task-scoped skills follow the [flutter/skills](https://github.com/flutter/skills) naming pattern (`glueful-<verb>-<noun>`) and should be verified against the framework source before merging — see [Versioning & accuracy](#versioning--accuracy).

## Versioning & accuracy

Skills target framework **conventions**, which can change between releases. When the framework changes a convention (e.g. a method signature or a default), the affected skill must be updated — stale guidance spreads wrong patterns. Each skill notes the framework version where version-sensitive behavior applies. Issues and PRs welcome.

## License

MIT © Glueful
