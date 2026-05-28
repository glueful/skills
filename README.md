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
| [`glueful-write-migration`](skills/glueful-write-migration/SKILL.md) | Adding or changing database schema — `MigrationInterface` + the schema builder, idempotency guards, a working `down()`, and the version-safe `alterTable` form. |
| [`glueful-add-controller`](skills/glueful-add-controller/SKILL.md) | Building API endpoints — `BaseController`, reading/validating input, the authenticated-user context, and the `Response` envelope. |
| [`glueful-add-route`](skills/glueful-add-route/SKILL.md) | Registering routes — the fluent router, groups, attribute routing, middleware, and the builder-form rate limiting (not the ignored string form). |

### Roadmap

Planned task-scoped skills (following the [flutter/skills](https://github.com/flutter/skills) naming pattern), to be added as each is verified against the framework:

- `glueful-build-orm-query` — Model statics, relations, eager loading, query result caching
- `glueful-create-extension` — `ServiceProvider`, `services()`, `register()`/`boot()`, command discovery
- `glueful-write-test` — the SQLite-backed `Connection` test pattern

Each skill is plain Markdown (`SKILL.md` with `name` + `description` frontmatter), so it works with any agent SDK that reads the skills convention.

## Versioning & accuracy

Skills target framework **conventions**, which can change between releases. When the framework changes a convention (e.g. a method signature or a default), the affected skill must be updated — stale guidance spreads wrong patterns. Each skill notes the framework version where version-sensitive behavior applies. Issues and PRs welcome.

## License

MIT © Glueful
