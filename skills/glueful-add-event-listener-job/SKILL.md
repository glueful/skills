---
name: glueful-add-event-listener-job
description: Wire domain events, listeners/subscribers, and queued jobs in a Glueful app — defining a BaseEvent, dispatching it from a service, handling it with an EventSubscriberInterface subscriber registered via EventService::subscribe(), and deferring heavy work to a Queue Job pushed through QueueManager. Use when adding side effects (notifications, counters, webhooks, search indexing) to a feature. Glueful events are PSR-14 BaseEvent + subscribers, not Laravel events/listeners or facades.
---

# Events, Listeners & Jobs

Side effects in a Glueful app ride on **domain events**: a service dispatches an event, **subscribers** react synchronously, and anything heavy/slow is pushed to a **queued job**. This keeps services focused on the use-case and request latency low.

```
Service  ──dispatch──►  Event (BaseEvent)  ──►  Subscriber handler (sync)  ──push──►  Job (async, queue worker)
```

## 1. Define the event

Extend `Glueful\Events\Contracts\BaseEvent` and **call `parent::__construct()`** (it sets the event id/timestamp/metadata). Carry the data as readonly constructor props:

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

## 2. Dispatch it from a service

Services emit events (not controllers — see `glueful-architect-app`). Inject `EventService` and dispatch:

```php
use Glueful\Events\EventService;

private EventService $eventService;
// ...
$this->eventService->dispatch(new ArticleCreated($article, $authorUuid));
```

(An event may instead `use Glueful\Events\Traits\Dispatchable;` to allow `ArticleCreated::dispatch($context, $article, $authorUuid)` — but injecting `EventService` is the common, testable form.)

## 3. Handle it — a subscriber

Implement `Glueful\Events\EventSubscriberInterface`: `getSubscribedEvents()` maps each event class to a handler method name; each handler takes the event. Inject whatever the side effect needs (e.g. `QueueManager`):

```php
namespace App\Listeners;

use App\Events\Article\ArticleCreated;
use App\Jobs\UpdateSearchCountersJob;
use Glueful\Events\EventSubscriberInterface;
use Glueful\Queue\QueueManager;

final class SearchCounterListener implements EventSubscriberInterface
{
    public function __construct(private readonly QueueManager $queue) {}

    /** @return array<class-string, string> event class => handler method */
    public static function getSubscribedEvents(): array
    {
        return [
            ArticleCreated::class => 'onArticleCreated',
            // ArticleUpdated::class => 'onArticleUpdated', ...
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

### Register the subscriber

In an app event provider's `register()`/`boot()`, subscribe it (this is the current framework pattern):

```php
// app/Providers/EventServiceProvider.php  (listed in config/serviceproviders.php)
$events = app($context, \Glueful\Events\EventService::class);
$events->subscribe(\App\Listeners\SearchCounterListener::class);
$events->subscribe(\App\Listeners\WebhookEventListener::class);
```

A subscriber that isn't `subscribe()`d never fires — this is the #1 "my listener didn't run" cause. (Simple static listener maps can alternatively live in `config/events.php`, but subscribers via `EventService::subscribe()` are the richer, current approach.)

## 4. Defer heavy work to a job

Extend `Glueful\Queue\Job`; the constructor receives the `$data` you pushed (read it back with `$this->getData()`); do the work in `handle()`. To change retries/timeout, **override `getMaxAttempts()`/`getTimeout()`** — the base returns `3`/`60` and ignores any `$maxAttempts`/`$timeout` properties:

```php
namespace App\Jobs;

use Glueful\Queue\Job;

class UpdateSearchCountersJob extends Job
{
    public function handle(): void
    {
        $payload     = $this->getData();              // accessor — NOT $this->data
        $articleUuid = $payload['article_uuid'] ?? null;
        // ... recompute / index / increment counters ...
    }

    // Tune retries/timeout by OVERRIDING the methods (the base returns 3 / 60).
    // Setting $maxAttempts / $timeout PROPERTIES does nothing — the base getters ignore them.
    public function getMaxAttempts(): int { return 3; }
    public function getTimeout(): int { return 30; }

    public function failed(\Exception $e): void
    {
        // optional: log / alert on permanent failure
    }
}
```

Dispatch via `QueueManager::push(jobClass, $data, ?queue)` (or `later(...)` to delay). Workers run them:

```bash
php glueful queue:work            # process queued jobs
```

**Job payloads must be serializable data** — pass scalars/arrays (uuids, ids), not live objects/models. The job re-fetches what it needs in `handle()`.

## Sync vs async — what goes where

- **In the subscriber (synchronous, in the request path):** cheap, fast, must-happen-now work (set a flag, dispatch a follow-on event).
- **In a job (async, off the request path):** anything slow or failure-prone — sending email/push, calling webhooks, re-indexing search, recomputing counters. A typical app queues notifications, search-counter updates, and webhook deliveries this way.

If a side effect can fail or take time, it belongs in a job, not inline — otherwise it slows (or breaks) the user's request.

## Gotchas

- **`parent::__construct()` is required** in every event — omitting it breaks event id/timestamp/metadata.
- **Subscribers must be registered** via `EventService::subscribe(...)` in a provider, or they silently never run.
- **`getSubscribedEvents()` maps `EventClass::class => 'methodName'`**; the handler method name must match exactly and accept the event.
- **Jobs carry data, not objects** — push uuids/arrays; re-load in `handle()`.
- **Retries/timeout are methods, not properties** — override `getMaxAttempts()`/`getTimeout()`; a `protected $maxAttempts` property is silently ignored (the base getters return `3`/`60`).
- **Events are dispatched from services**, not controllers (keeps HTTP thin).

## Checklist

- [ ] Event extends `BaseEvent` and calls `parent::__construct()`; data as readonly props.
- [ ] Dispatched from a service via injected `EventService::dispatch(...)`.
- [ ] Subscriber implements `EventSubscriberInterface` with `getSubscribedEvents()` + matching `on*` handlers; registered via `EventService::subscribe()` in a provider.
- [ ] Heavy/failure-prone side effects pushed to a `Queue\Job` (`QueueManager::push(Job::class, $data)`) with serializable data; light work inline.
- [ ] Job extends `Glueful\Queue\Job`, implements `handle()` (+ `failed()` if needed); verified with `php glueful queue:work`.
