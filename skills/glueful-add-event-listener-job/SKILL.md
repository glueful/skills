---
name: glueful-add-event-listener-job
description: Wire domain events, listeners/subscribers, and queued jobs in a Glueful app — defining a BaseEvent, dispatching it from a service, handling it with an EventSubscriberInterface subscriber or an AsListener-attributed class registered via EventService::subscribe(), deferring heavy work to a Queue Job pushed through QueueManager, and scheduling recurring jobs. Use when adding side effects (notifications, counters, webhooks, search indexing) to a feature. Glueful events are PSR-14 BaseEvent + subscribers, not Laravel events/listeners or facades.
---

# Events, Listeners & Jobs

Side effects in a Glueful app ride on **domain events**: a service dispatches an event, **listeners** react synchronously, and anything heavy/slow is pushed to a **queued job**. This keeps services focused on the use-case and request latency low.

```
Service  ──dispatch──►  Event (BaseEvent)  ──►  Listener (sync)  ──push──►  Job (async, queue worker)
```

## 1. Define the event

Extend `Glueful\Events\Contracts\BaseEvent` and **call `parent::__construct()`** (it sets the event id/timestamp). Carry the data as readonly constructor props:

```php
namespace App\Events\Article;

use Glueful\Events\Contracts\BaseEvent;

final class ArticleCreated extends BaseEvent
{
    /** @param array<string,mixed> $article */
    public function __construct(
        public readonly array $article,
        public readonly string $authorUuid,
    ) {
        parent::__construct();   // REQUIRED — don't omit
    }

    public function getArticleUuid(): string
    {
        return $this->article['uuid'] ?? '';
    }
}
```

`php glueful event:create Article/ArticleCreated` scaffolds one under `app/Events/` (it appends an `Event` suffix to the class name). Framework built-in events (`Glueful\Events\Auth\*`, `Http\*`, `Database\*`, `Webhook\*` …) expose **getters** such as `getRequest()` / `getUserUuid()`, not public props.

## 2. Dispatch it from a service

Services emit events (not controllers — see `glueful-architect-app`). Inject `EventService` and dispatch:

```php
use Glueful\Events\EventService;

public function __construct(private readonly EventService $events) {}
// ...
$this->events->dispatch(new ArticleCreated($article, $authorUuid));
```

- **`dispatch()` is fault-isolated**: a listener that throws is logged (`error_log`) and skipped; the remaining listeners still run and `dispatch()` never throws.
- **`dispatchOrFail()`** rethrows the first failing listener's original exception and stops the chain — for callers that must observe/redrive failure (e.g. a durable inbound-webhook store). Delivery becomes at-least-once, so those listeners must be idempotent.
- An event may instead `use Glueful\Events\Traits\Dispatchable;` for `ArticleCreated::dispatch($context, $article, $authorUuid)` — but injecting `EventService` is the common, testable form.

## 3. Handle it — a subscriber or an `#[AsListener]` class

**Subscriber:** implement `Glueful\Events\EventSubscriberInterface`. `getSubscribedEvents()` maps each event class to a handler method name, or to `[method, priority]` (higher priority runs first). Constructor dependencies are autowired:

```php
namespace App\Listeners;

use App\Events\Article\ArticleCreated;
use App\Jobs\UpdateSearchCountersJob;
use Glueful\Events\EventSubscriberInterface;
use Glueful\Queue\QueueManager;

final class SearchCounterListener implements EventSubscriberInterface
{
    public function __construct(private readonly QueueManager $queue) {}

    /** @return array<string, string|array{0:string,1?:int}> */
    public static function getSubscribedEvents(): array
    {
        return [
            ArticleCreated::class => 'onArticleCreated',
            // ArticleDeleted::class => ['onArticleDeleted', 10],
        ];
    }

    public function onArticleCreated(ArticleCreated $event): void
    {
        // Light work can happen inline; heavy work → queue a job:
        $this->queue->push(UpdateSearchCountersJob::class, [
            'article_uuid' => $event->getArticleUuid(),
            'action'       => 'created',
        ]);
    }
}
```

**Attribute form (framework ≥ 1.91):** put `#[AsListener(Event::class, priority: 0)]` (`Glueful\Events\Attributes\AsListener`) on public methods — or on the class with `method:` (default `__invoke`). It is read by the same `subscribe()` call; there is **no directory scanning**. A class may combine both forms. `php glueful event:listener Name --event=...` scaffolds this form in `app/Events/Listeners/`.

```php
final class OrderListeners
{
    #[AsListener(OrderPlaced::class, priority: 10)]
    public function reserveStock(OrderPlaced $event): void { /* ... */ }
}
```

### Register it

Subscribe in the `boot()` of an app provider. App providers are listed in `config/serviceproviders.php` → `enabled` (string FQCNs, no `::class`), and to get a `boot()` call the class must **extend `Glueful\Extensions\ServiceProvider`** — a plain `services()`-only class like the skeleton's `AppServiceProvider` is never booted:

```php
// app/Providers/EventServiceProvider.php — add 'App\\Providers\\EventServiceProvider' to config/serviceproviders.php
namespace App\Providers;

use App\Listeners\OrderListeners;
use App\Listeners\SearchCounterListener;
use Glueful\Bootstrap\ApplicationContext;
use Glueful\Events\EventService;
use Glueful\Extensions\ServiceProvider;

final class EventServiceProvider extends ServiceProvider
{
    public function boot(ApplicationContext $context): void
    {
        $events = $this->app->get(EventService::class);
        $events->subscribe(SearchCounterListener::class);
        $events->subscribe(OrderListeners::class);
    }
}
```

A class that isn't `subscribe()`d never fires — the #1 "my listener didn't run" cause. There is **no config-file listener map** (`config/events.php` only toggles core listeners). `subscribe()` keeps container-registered classes lazy; an unregistered class is autowired immediately, and one that can't be built — or has neither `getSubscribedEvents()` nor `#[AsListener]` — throws `InvalidArgumentException` at `subscribe()` (i.e. at boot). For one-offs use `$events->addListener(Event::class, $callable | '@serviceId:method', $priority)`.

> **Outbound webhooks** need no listener: mark the event `#[Webhookable(event: 'article.created')]` (`Glueful\Api\Webhooks\Attributes\Webhookable`) or use `DispatchesWebhooks`; the framework's webhook bridge (since 1.91) delivers it to subscribed endpoints. Don't also register `WebhookEventListener` yourself — each event would send twice.

## 4. Defer heavy work to a job

Extend `Glueful\Queue\Job`. The worker constructs it with the pushed `$data` (read it back with `$this->getData()`) and the app context (**`$this->context`** — there is no `$context` variable inside a job). Do the work in `handle()`:

```php
namespace App\Jobs;

use Glueful\Queue\Job;
use Psr\Log\LoggerInterface;

class UpdateSearchCountersJob extends Job
{
    public function handle(): void
    {
        $payload     = $this->getData();              // accessor — NOT $this->data
        $articleUuid = $payload['article_uuid'] ?? null;
        $db = db($this->context);                     // helpers take $this->context
        // ... recompute / index / increment counters ...
        // Transient failure? $this->release(300);    // re-queue with a delay
    }

    public function failed(\Exception $e): void       // optional: runs on permanent failure
    {
        app($this->context, LoggerInterface::class)->error('Counter update failed', ['error' => $e->getMessage()]);
    }
}
```

`php glueful scaffold:job UpdateSearchCounters` generates one in `app/Jobs/`. If a job keeps a constructor, its signature must stay `__construct(array $data = [], ?ApplicationContext $context = null)` and pass **both** to `parent::__construct()` — otherwise `$this->context` is null.

Dispatch via `QueueManager::push(jobClass, $data, ?queue, ?connection)` (or `later($delaySeconds, jobClass, $data, …)`). **Retries:** the worker allows `min($data['maxAttempts'] ?? 3, queue:work --tries)` attempts — to change it, push `'maxAttempts' => 5` in the data (and run workers with `--tries` ≥ that). Since 1.91.3 a `getMaxAttempts()` override on the job class counts too (a pushed `maxAttempts` wins; a `$maxAttempts` property is never read); before 1.91.3 only the pushed value did. `queue:work --timeout` is enforced since 1.91.3 (with pcntl: the job is released/failed and the worker exits, so run workers under a process manager); there is no per-job timeout.

**Job payloads must be serializable data** — pass scalars/arrays (uuids, ids), not live objects/models. The job re-fetches what it needs in `handle()`.

### Running and operating workers

```bash
php glueful queue:work                        # loop (--queue=a,b --connection= --sleep= --memory= --max-jobs= --max-runtime= --tries=)
php glueful queue:work --stop-when-empty      # drain, then exit   (--once = ONE job, then exit)
php glueful queue:restart                     # deploy step: workers exit after their current job
php glueful queue:failed [--queue=] [--json]  # inspect failures
php glueful queue:retry <uuid>… | --all [--queue=]
php glueful queue:forget <uuid>   /   queue:flush [--queue=]
```

Supervised worker fleets / autoscaling come from the `glueful/queue-ops` extension, not core.

### Queue security

- **Handler classes must implement `JobInterface`** (extending `Glueful\Queue\Job` does). A plain class with a `handle()` method is refused at dequeue ("Refusing to run job class …") and the job fails. Same gate for scheduler `handler_class` entries.
- **Persisted payloads are HMAC-signed with `APP_KEY`** (`QUEUE_PAYLOAD_SIGNING`, default on; verified before any handler code runs). Keep `APP_KEY` identical on every process that pushes or works jobs (without one, signing is inert); never hand-edit queue/scheduler rows — the signature breaks and the job fails terminally. When upgrading with pre-signing rows still queued, set `QUEUE_REQUIRE_SIGNED_PAYLOADS=false` temporarily while they drain.

## 5. Recurring work — scheduled jobs

Declare recurring jobs in `config/schedule.php` → `jobs` (each: `name`, `schedule` cron, `handler_class`, optional `parameters`, `enabled`, `persistence`, `description`). The handler must extend `Glueful\Queue\Job` (`parameters` arrive via `getData()`); it runs **inline** in the scheduler process — there is no per-job `queue` / `timeout` / `retry_attempts`. Run `php glueful queue:scheduler run` from cron every minute (or `queue:scheduler work` as a daemon; `list` / `status` to inspect).

> **Config lists replace, not append (since 1.86).** An app's `schedule.jobs` list **replaces** the framework's, so copy the defaults you still want (`session_cleaner`, `queue_maintenance`, `scheduled_notifications`, `webhook_cleanup`, `blob_purge`, …) into your list alongside your own.

## Sync vs async — what goes where

- **In the listener (synchronous, in the request path):** cheap, fast, must-happen-now work (set a flag, dispatch a follow-on event).
- **In a job (async, off the request path):** anything slow or failure-prone — sending email/push, calling webhooks, re-indexing search, recomputing counters.

If a side effect can fail or take time, it belongs in a job — inline it slows the request, and a throwing listener's work is lost apart from a log line.

## Gotchas

- **`parent::__construct()` is required** in every event.
- **Listeners must be `subscribe()`d** in a booted provider (extends `Glueful\Extensions\ServiceProvider`, listed in `config/serviceproviders.php`); `#[AsListener]` alone does nothing.
- **`getSubscribedEvents()` maps `EventClass::class => 'method'` or `['method', priority]`**; the method must be public and accept the event.
- **Jobs use `$this->context`**, carry data not objects, and re-load in `handle()`.
- **Retry count = pushed `maxAttempts`, else the job's `getMaxAttempts()` (since 1.91.3; default 3), capped by `--tries`.**
- **Don't "simplify" a job to a plain `handle()` class** — `JobInterface` is required; extend `Glueful\Queue\Job`.
- **`--once` runs one job**; use `--stop-when-empty` to drain.
- **Events are dispatched from services**, not controllers (keeps HTTP thin).

## Checklist

- [ ] Event extends `BaseEvent` and calls `parent::__construct()`; data as readonly props.
- [ ] Dispatched from a service via injected `EventService::dispatch(...)` (`dispatchOrFail()` only when failure must propagate, with idempotent listeners).
- [ ] Listener implements `EventSubscriberInterface` or uses `#[AsListener]`, and is `subscribe()`d in a provider's `boot()`.
- [ ] Heavy/failure-prone side effects pushed to a `Queue\Job` (`QueueManager::push(Job::class, $data)`) with serializable data (+ `maxAttempts` if not 3).
- [ ] Job extends `Glueful\Queue\Job`, uses `$this->context` / `getData()`, implements `handle()` (+ `failed()` if needed); verified with `php glueful queue:work --stop-when-empty`; `queue:restart` in the deploy.
- [ ] Recurring work in `config/schedule.php`, with the framework's default jobs re-included.
