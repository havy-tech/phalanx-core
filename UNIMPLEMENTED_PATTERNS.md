# Unimplemented Patterns from Original Design

These patterns were documented in the original Convoy design conversations but haven't been implemented in v2. Each represents a capability that extends the current Task/Scope system.

---

## 1. FiberLocal Storage

**Problem:** PHP has no native fiber-local storage. Code deep in the call stack can't access the current Scope without threading it through every function signature.

**Solution:** WeakMap keyed by Fiber instances provides implicit context access.

```php
<?php

declare(strict_types=1);

namespace Convoy\Task;

use Closure;
use Fiber;
use WeakMap;

final class FiberLocal
{
    private static WeakMap $storage;

    public function __construct(
        public readonly Scope $scope,
        public readonly Dispatchable $task,
        private array $values = [],
    ) {
        self::$storage ??= new WeakMap();
    }

    public function set(string $key, mixed $value): void
    {
        $this->values[$key] = $value;
    }

    public function get(string $key, mixed $default = null): mixed
    {
        return $this->values[$key] ?? $default;
    }

    public function has(string $key): bool
    {
        return array_key_exists($key, $this->values);
    }

    /**
     * Lazy-computed value -- factory runs once per fiber, result cached
     */
    public function lazy(string $key, Closure $factory): mixed
    {
        if (!array_key_exists($key, $this->values)) {
            $this->values[$key] = $factory($this->scope);
        }
        return $this->values[$key];
    }

    /**
     * Get current fiber's local storage
     */
    public static function current(): ?self
    {
        $fiber = Fiber::getCurrent();
        if ($fiber === null) {
            return null;
        }
        return self::$storage[$fiber] ?? null;
    }

    /**
     * Register FiberLocal for current fiber
     */
    public static function register(self $local): void
    {
        self::$storage ??= new WeakMap();
        $fiber = Fiber::getCurrent();
        if ($fiber !== null) {
            self::$storage[$fiber] = $local;
        }
    }

    /**
     * Unregister current fiber's storage
     */
    public static function unregister(): void
    {
        $fiber = Fiber::getCurrent();
        if ($fiber !== null && isset(self::$storage[$fiber])) {
            unset(self::$storage[$fiber]);
        }
    }
}
```

### Integration with ExecutionScope

```php
<?php

public function execute(Dispatchable $task): mixed
{
    $fiber = Fiber::getCurrent();
    $local = new FiberLocal($this, $task);

    if ($fiber !== null) {
        FiberLocal::register($local);
    }

    try {
        return $this->applyBehaviorPipeline($task);
    } finally {
        if ($fiber !== null) {
            FiberLocal::unregister();
        }
    }
}
```

### Usage Anywhere in Call Stack

```php
<?php

// Deep in library code, no $scope parameter available
$ctx = FiberLocal::current();
if ($ctx !== null) {
    $requestId = $ctx->lazy('request_id', static fn() => Uuid::v7());
    $logger = $ctx->scope->service(Logger::class);
    $logger->info('Processing', ['request' => $requestId]);
}
```

**Key insight:** WeakMap keyed by Fiber means storage is automatically garbage collected when the fiber terminates. No cleanup required.

---

## 2. Task Templates via Lazy Ghosts

**Problem:** Defining common task patterns (HTTP GET with retry, DB query with timeout) requires repetitive boilerplate.

**Solution:** PHP 8.4 lazy ghosts defer Task instantiation until first invocation. Templates define the pattern once; actual Task objects created on demand.

```php
<?php

declare(strict_types=1);

namespace Convoy\Task;

use Closure;
use ReflectionClass;

final class TaskTemplate
{
    /** @var array<string, array{factory: Closure, config: TaskConfig, instance: ?Task}> */
    private static array $templates = [];

    /**
     * Define a lazy task template
     *
     * Factory runs only when template is actually invoked
     */
    public static function define(string $name, Closure $factory, TaskConfig $config): void
    {
        self::$templates[$name] = [
            'factory' => $factory,
            'config' => $config,
            'instance' => null,
        ];
    }

    /**
     * Get a lazy ghost of the task template
     *
     * The Task is not instantiated until __invoke() is called
     */
    public static function get(string $name): Task
    {
        if (!isset(self::$templates[$name])) {
            throw new \InvalidArgumentException("Unknown task template: {$name}");
        }

        $template = &self::$templates[$name];

        // Return cached instance if exists
        if ($template['instance'] !== null) {
            return $template['instance'];
        }

        $reflector = new ReflectionClass(Task::class);

        // Create lazy ghost -- Task constructor doesn't run until property/method access
        $template['instance'] = $reflector->newLazyGhost(
            static function (Task $ghost) use ($template): void {
                $realTask = ($template['factory'])();

                // Copy properties from real task to ghost
                $taskReflector = new ReflectionClass(Task::class);
                foreach ($taskReflector->getProperties() as $prop) {
                    if (!$prop->isStatic()) {
                        $prop->setAccessible(true);
                        $prop->setValue($ghost, $prop->getValue($realTask));
                    }
                }
            }
        );

        return $template['instance'];
    }

    /**
     * Check if template exists
     */
    public static function has(string $name): bool
    {
        return isset(self::$templates[$name]);
    }

    /**
     * Clear all templates (for testing)
     */
    public static function clear(): void
    {
        self::$templates = [];
    }
}
```

### Registration at Boot Time

```php
<?php

// Bootstrap -- no Task objects created yet
TaskTemplate::define(
    name: 'http.get',
    factory: static fn() => new Task(
        static fn(Scope $s) => $s->service(HttpClient::class)->get(
            FiberLocal::current()?->get('url')
                ?? throw new \RuntimeException('URL required in FiberLocal')
        ),
    ),
    config: new TaskConfig(
        name: 'HTTP GET',
        timeout: 30.0,
        retry: RetryPolicy::exponential(3, 100, 5000),
    ),
);

TaskTemplate::define(
    name: 'db.query',
    factory: static fn() => new Task(
        static fn(Scope $s) => $s->service(Database::class)->query(
            FiberLocal::current()?->get('sql'),
            FiberLocal::current()?->get('params', []),
        ),
    ),
    config: new TaskConfig(
        name: 'DB Query',
        timeout: 10.0,
        pool: Pool::Database,
    ),
);
```

### Usage

```php
<?php

// Gets lazy ghost -- Task not instantiated yet
$httpGet = TaskTemplate::get('http.get');

// Set context for the task
FiberLocal::current()?->set('url', '/api/users/42');

// NOW Task instantiates (lazy ghost triggers), THEN executes
$result = $scope->execute($httpGet);
```

**Key insight:** Lazy ghosts avoid instantiation cost for templates that may never be used. The factory closure captures the pattern; the ghost defers it.

---

## 3. TaskConfig Decorator Chain

**Problem:** v2 uses behavior interfaces (Retryable, HasTimeout) which work but require implementing interfaces. Sometimes you want ad-hoc behavior composition without class definitions.

**Solution:** TaskConfig accumulates decorators. `__invoke()` wraps the work closure in nested behavior layers.

```php
<?php

declare(strict_types=1);

namespace Convoy\Task;

use Closure;

final readonly class TaskConfig
{
    /**
     * @param array<Closure(Scope, Closure): mixed> $decorators
     * @param array<Closure(Scope, Dispatchable): void> $before
     * @param array<Closure(Scope, Dispatchable, mixed): void> $after
     * @param array<Closure(Scope, Dispatchable, \Throwable): void> $onError
     * @param array<Closure(): void> $cleanup
     */
    public function __construct(
        public string $name = '',
        public int $priority = 0,
        public ?UnitEnum $pool = null,
        public ?RetryPolicy $retry = null,
        public ?float $timeout = null,
        public ?int $concurrencyLimit = null,
        public bool $trace = true,
        public array $decorators = [],
        public array $before = [],
        public array $after = [],
        public array $onError = [],
        public array $cleanup = [],
        public array $tags = [],
    ) {}

    public function with(
        ?string $name = null,
        ?int $priority = null,
        ?UnitEnum $pool = null,
        ?RetryPolicy $retry = null,
        ?float $timeout = null,
        ?int $concurrencyLimit = null,
        ?bool $trace = null,
        ?array $decorators = null,
        ?array $before = null,
        ?array $after = null,
        ?array $onError = null,
        ?array $cleanup = null,
        ?array $tags = null,
    ): self {
        return new self(
            name: $name ?? $this->name,
            priority: $priority ?? $this->priority,
            pool: $pool ?? $this->pool,
            retry: $retry ?? $this->retry,
            timeout: $timeout ?? $this->timeout,
            concurrencyLimit: $concurrencyLimit ?? $this->concurrencyLimit,
            trace: $trace ?? $this->trace,
            decorators: $decorators ?? $this->decorators,
            before: $before ?? $this->before,
            after: $after ?? $this->after,
            onError: $onError ?? $this->onError,
            cleanup: $cleanup ?? $this->cleanup,
            tags: $tags ?? $this->tags,
        );
    }

    /**
     * Merge another config (useful for combining templates)
     */
    public function merge(self $other): self
    {
        return new self(
            name: $other->name ?: $this->name,
            priority: $other->priority ?: $this->priority,
            pool: $other->pool ?? $this->pool,
            retry: $other->retry ?? $this->retry,
            timeout: $other->timeout ?? $this->timeout,
            concurrencyLimit: $other->concurrencyLimit ?? $this->concurrencyLimit,
            trace: $other->trace && $this->trace,
            decorators: [...$this->decorators, ...$other->decorators],
            before: [...$this->before, ...$other->before],
            after: [...$this->after, ...$other->after],
            onError: [...$this->onError, ...$other->onError],
            cleanup: [...$this->cleanup, ...$other->cleanup],
            tags: [...$this->tags, ...$other->tags],
        );
    }
}
```

### Decorator Chain Builder

```php
<?php

final class ConfigurableTask implements Dispatchable
{
    public function __construct(
        private readonly Closure $work,
        public readonly TaskConfig $config = new TaskConfig(),
    ) {
        // Enforce static closure
        $rf = new \ReflectionFunction($work);
        if ($rf->getClosureThis() !== null) {
            throw new \InvalidArgumentException(
                'Task closure must be static to prevent reference cycles'
            );
        }
    }

    public function __invoke(Scope $scope): mixed
    {
        // Build execution pipeline from config
        $pipeline = $this->work;

        // 1. Wrap with timeout if configured
        if ($this->config->timeout !== null) {
            $timeout = $this->config->timeout;
            $inner = $pipeline;
            $pipeline = static fn(Scope $s) => $s->timeout($timeout, new Task($inner));
        }

        // 2. Wrap with retry if configured
        if ($this->config->retry !== null) {
            $policy = $this->config->retry;
            $inner = $pipeline;
            $pipeline = static fn(Scope $s) => $s->retry(new Task($inner), $policy);
        }

        // 3. Wrap with tracing if enabled
        if ($this->config->trace && $this->config->name !== '') {
            $name = $this->config->name;
            $inner = $pipeline;
            $pipeline = static fn(Scope $s) => $s->traced($name, new Task($inner));
        }

        // 4. Apply custom decorators (in reverse for correct nesting)
        foreach (array_reverse($this->config->decorators) as $decorator) {
            $inner = $pipeline;
            $pipeline = static fn(Scope $s) => $decorator($s, $inner);
        }

        // 5. Wrap with error handling
        $inner = $pipeline;
        $onError = $this->config->onError;
        $cleanup = $this->config->cleanup;

        $pipeline = static function(Scope $s) use ($inner, $onError, $cleanup): mixed {
            try {
                return $inner($s);
            } catch (\Throwable $e) {
                foreach ($onError as $handler) {
                    $handler($s, $e);
                }
                throw $e;
            } finally {
                foreach ($cleanup as $fn) {
                    $fn();
                }
            }
        };

        // 6. Run before hooks
        foreach ($this->config->before as $hook) {
            $hook($scope, $this);
        }

        // Execute pipeline
        $result = $pipeline($scope);

        // 7. Run after hooks
        foreach ($this->config->after as $hook) {
            $hook($scope, $this, $result);
        }

        return $result;
    }
}
```

### Usage

```php
<?php

$task = new ConfigurableTask(
    work: static fn(Scope $s) => $s->service(HttpClient::class)->get('/api/data'),
    config: new TaskConfig(
        name: 'FetchData',
        timeout: 5.0,
        retry: RetryPolicy::exponential(3),
        before: [
            static fn(Scope $s, $t) => $s->service(Logger::class)->info('Starting fetch'),
        ],
        after: [
            static fn(Scope $s, $t, $result) =>
                $s->service(Metrics::class)->recordLatency('fetch_data', hrtime(true)),
        ],
        decorators: [
            // Custom auth header injection
            static fn(Scope $s, Closure $next) => $s->withAttribute(
                'auth_token',
                $s->service(Auth::class)->getToken()
            )->execute(new Task($next)),
        ],
    ),
);
```

**Key insight:** Decorator composition is declarative. Order matters: timeout wraps retry wraps trace wraps custom decorators wraps work.

---

## 4. Affinity-Based Task Scheduler

**Problem:** Running 100 concurrent HTTP requests, 50 DB queries, and 20 Redis operations simultaneously exhausts connection pools.

**Solution:** Tasks declare their "affinity" (which pool they use). Scheduler enforces per-affinity concurrency limits.

```php
<?php

declare(strict_types=1);

namespace Convoy\Task;

use Closure;
use Fiber;
use SplPriorityQueue;
use WeakMap;
use function React\Async\async;
use function React\Async\await;
use function React\Promise\any;

final class AffinityScheduler implements Dispatchable
{
    private SplPriorityQueue $queue;
    private WeakMap $results;
    private array $runningByAffinity = [];

    /**
     * @param Dispatchable[] $tasks
     * @param array<string, int> $affinityLimits  e.g., ['Http' => 50, 'Database' => 10]
     */
    public function __construct(
        private readonly array $tasks,
        private readonly array $affinityLimits = [],
        private readonly int $defaultLimit = 100,
    ) {
        $this->queue = new SplPriorityQueue();
        $this->queue->setExtractFlags(SplPriorityQueue::EXTR_DATA);
        $this->results = new WeakMap();
    }

    public function __invoke(Scope $scope): array
    {
        // Queue tasks by priority
        foreach ($this->tasks as $index => $task) {
            $priority = $task instanceof HasPriority ? $task->priority : 0;
            $this->queue->insert(['task' => $task, 'index' => $index], $priority);
        }

        // Process queue respecting affinity limits
        $this->processQueue($scope);

        // Collect results in original order
        $results = [];
        foreach ($this->tasks as $index => $task) {
            if (isset($this->results[$task])) {
                $results[$index] = $this->results[$task];
            }
        }

        return $results;
    }

    private function processQueue(Scope $scope): void
    {
        $pending = [];

        while (!$this->queue->isEmpty() || !empty($pending)) {
            // Start tasks up to affinity limits
            while (!$this->queue->isEmpty()) {
                $item = $this->queue->top();
                $task = $item['task'];

                // Resolve affinity from UsesPool interface or default
                $affinity = $task instanceof UsesPool
                    ? $task->pool->name
                    : 'default';

                $limit = $this->affinityLimits[$affinity] ?? $this->defaultLimit;
                $currentRunning = $this->runningByAffinity[$affinity] ?? 0;

                if ($currentRunning >= $limit) {
                    break; // At capacity for this affinity
                }

                $this->queue->extract();
                $this->runningByAffinity[$affinity] = $currentRunning + 1;

                $results = $this->results;
                $running = &$this->runningByAffinity;

                // Launch fiber
                $pending[] = [
                    'task' => $task,
                    'affinity' => $affinity,
                    'promise' => async(static function() use ($scope, $task, $results): mixed {
                        $result = $scope->execute($task);
                        $results[$task] = $result;
                        return $result;
                    })(),
                ];
            }

            if (empty($pending)) {
                break;
            }

            // Wait for any to complete
            $promises = array_column($pending, 'promise');
            await(any($promises));

            // Update running counts and remove completed
            $running = &$this->runningByAffinity;
            $pending = array_filter($pending, static function($item) use (&$running): bool {
                // Check if promise resolved (React Promise API)
                $state = null;
                $item['promise']->then(
                    static function() use (&$state) { $state = 'fulfilled'; },
                    static function() use (&$state) { $state = 'rejected'; }
                );

                if ($state !== null) {
                    $running[$item['affinity']]--;
                    return false;
                }
                return true;
            });
        }
    }
}
```

### Usage

```php
<?php

use Convoy\Task\Pool;
use App\Task\AppPool;

$scheduler = new AffinityScheduler(
    tasks: [
        new FetchUser(1),           // UsesPool -> Pool::Http
        new FetchUser(2),           // UsesPool -> Pool::Http
        new SaveOrder($order),      // UsesPool -> Pool::Database
        new ProcessPayment($data),  // UsesPool -> AppPool::PaymentGateway
        new CacheUser($user),       // UsesPool -> Pool::Redis
        // ... 100 more tasks
    ],
    affinityLimits: [
        Pool::Http->name => 50,              // max 50 concurrent HTTP
        Pool::Database->name => 10,          // max 10 concurrent DB
        Pool::Redis->name => 20,             // max 20 concurrent Redis
        AppPool::PaymentGateway->name => 5,  // max 5 concurrent payments
    ],
);

$results = $scope->execute($scheduler);
```

**Key insight:** Tasks don't know about limits. They just declare their pool via `UsesPool` interface. Scheduler reads pool, applies limits. Separation of concerns.

---

## 5. AI Tool Registry

**Problem:** AI agents need to call PHP functions as "tools". Each tool has an OpenAI-compatible schema and a Task implementation.

**Solution:** Registry maps tool names to Tasks. Schema generation is declarative. Tool execution uses FiberLocal for argument passing.

```php
<?php

declare(strict_types=1);

namespace Convoy\Task\AI;

use Convoy\Scope;
use Convoy\Task\Dispatchable;
use Convoy\Task\FiberLocal;
use Convoy\Task\Task;
use Convoy\Task\TaskConfig;

final class ToolRegistry
{
    /** @var array<string, array{task: Dispatchable, schema: array}> */
    private array $tools = [];

    /**
     * Register a tool with its Task implementation and OpenAI schema
     */
    public function register(string $name, Dispatchable $tool, array $schema): self
    {
        $this->tools[$name] = [
            'task' => $tool,
            'schema' => $schema,
        ];
        return $this;
    }

    /**
     * Get all tool schemas for OpenAI API
     */
    public function getSchemas(): array
    {
        return array_map(
            static fn(array $tool): array => $tool['schema'],
            $this->tools,
        );
    }

    /**
     * Get tool names
     */
    public function getToolNames(): array
    {
        return array_keys($this->tools);
    }

    /**
     * Execute a tool by name with given arguments
     */
    public function execute(string $name, array $args, Scope $scope): mixed
    {
        if (!isset($this->tools[$name])) {
            throw new \InvalidArgumentException("Unknown tool: {$name}");
        }

        // Set args in fiber-local context for tool to access
        $ctx = FiberLocal::current();
        if ($ctx !== null) {
            $ctx->set('tool_args', $args);
            $ctx->set('tool_name', $name);
        }

        return $scope->execute($this->tools[$name]['task']);
    }

    /**
     * Check if tool exists
     */
    public function has(string $name): bool
    {
        return isset($this->tools[$name]);
    }
}
```

### Tool Definition

```php
<?php

$registry = new ToolRegistry();

// Database search tool
$registry->register(
    name: 'search_database',
    tool: new Task(
        static function(Scope $s): array {
            $args = FiberLocal::current()?->get('tool_args') ?? [];

            return $s->service(Database::class)->search(
                table: $args['table'],
                query: $args['query'],
                limit: $args['limit'] ?? 10,
            );
        },
        new TaskConfig(
            name: 'AI.Tool.SearchDatabase',
            pool: Pool::Database,
            timeout: 5.0,
        ),
    ),
    schema: [
        'type' => 'function',
        'function' => [
            'name' => 'search_database',
            'description' => 'Search a database table for matching records',
            'parameters' => [
                'type' => 'object',
                'properties' => [
                    'table' => [
                        'type' => 'string',
                        'description' => 'The table name to search',
                    ],
                    'query' => [
                        'type' => 'string',
                        'description' => 'The search query',
                    ],
                    'limit' => [
                        'type' => 'integer',
                        'description' => 'Maximum results to return',
                    ],
                ],
                'required' => ['table', 'query'],
            ],
        ],
    ],
);

// Email sending tool
$registry->register(
    name: 'send_email',
    tool: new Task(
        static function(Scope $s): bool {
            $args = FiberLocal::current()?->get('tool_args') ?? [];

            return $s->service(EmailService::class)->send(
                to: $args['to'],
                subject: $args['subject'],
                body: $args['body'],
            );
        },
        new TaskConfig(
            name: 'AI.Tool.SendEmail',
            pool: Pool::Queue,
            timeout: 10.0,
        ),
    ),
    schema: [
        'type' => 'function',
        'function' => [
            'name' => 'send_email',
            'description' => 'Send an email to a recipient',
            'parameters' => [
                'type' => 'object',
                'properties' => [
                    'to' => ['type' => 'string', 'description' => 'Recipient email'],
                    'subject' => ['type' => 'string', 'description' => 'Email subject'],
                    'body' => ['type' => 'string', 'description' => 'Email body content'],
                ],
                'required' => ['to', 'subject', 'body'],
            ],
        ],
    ],
);
```

### AI Agent Loop

```php
<?php

final class AgentLoop implements Dispatchable
{
    public function __construct(
        private readonly ToolRegistry $registry,
        private readonly array $initialMessages,
        private readonly string $model = 'gpt-4',
    ) {}

    public function __invoke(Scope $scope): string
    {
        $client = $scope->service(OpenAIClient::class);
        $messages = $this->initialMessages;

        while (true) {
            $scope->throwIfCancelled();

            $response = $client->chat()->create([
                'model' => $this->model,
                'messages' => $messages,
                'tools' => $this->registry->getSchemas(),
            ]);

            $choice = $response->choices[0];

            if ($choice->finishReason === 'stop') {
                return $choice->message->content;
            }

            if ($choice->finishReason === 'tool_calls') {
                $toolCalls = $choice->message->toolCalls;

                // Execute tool calls concurrently
                $results = $scope->concurrent(
                    array_map(
                        fn($call) => new Task(
                            static fn(Scope $s) => $this->registry->execute(
                                $call->function->name,
                                json_decode($call->function->arguments, true),
                                $s,
                            )
                        ),
                        $toolCalls,
                    ),
                );

                // Add assistant message with tool calls
                $messages[] = $choice->message->toArray();

                // Add tool results
                foreach ($toolCalls as $i => $call) {
                    $messages[] = [
                        'role' => 'tool',
                        'tool_call_id' => $call->id,
                        'content' => json_encode($results[$i]),
                    ];
                }
            }
        }
    }
}
```

### Usage

```php
<?php

$agent = new AgentLoop(
    registry: $registry,
    initialMessages: [
        ['role' => 'system', 'content' => 'You are a helpful assistant with database and email access.'],
        ['role' => 'user', 'content' => 'Find all orders from last week and email a summary to admin@example.com'],
    ],
);

$response = $scope->execute($agent);
```

**Key insight:** Tools are Tasks. They get the full behavior pipeline (timeout, retry, tracing, pool limits). FiberLocal passes arguments without polluting function signatures.

---

## Integration Notes

### Dependencies Between Patterns

```
FiberLocal
    └── Required by: TaskTemplates, AI Tool Registry

TaskConfig Decorator Chain
    └── Alternative to: Behavior interfaces (Retryable, HasTimeout)
    └── Can coexist with current approach

AffinityScheduler
    └── Builds on: UsesPool interface, HasPriority interface
    └── Replaces: Basic TaskScheduler for complex workloads

AI Tool Registry
    └── Requires: FiberLocal
    └── Uses: Task, TaskConfig, concurrent execution
```

### Recommended Implementation Order

1. **FiberLocal** -- foundational, unlocks others
2. **TaskConfig decorator chain** -- optional, provides alternative to interfaces
3. **TaskTemplates** -- depends on FiberLocal
4. **AffinityScheduler** -- independent, uses existing interfaces
5. **AI Tool Registry** -- depends on FiberLocal

### PHP 8.4 Features Used

| Feature | Pattern |
|---------|---------|
| `WeakMap` | FiberLocal storage, AffinityScheduler results |
| `ReflectionClass::newLazyGhost()` | TaskTemplates |
| `array_any()` | Retry policy exception matching |
| Property hooks | TaskConfig computed values (if needed) |
| Asymmetric visibility | FiberLocal internal state |

---

## Summary

These patterns extend Convoy's capability without replacing the core Task/Scope/Behavior architecture:

| Pattern | Capability Added |
|---------|------------------|
| FiberLocal | Implicit context access anywhere in call stack |
| TaskTemplates | Reusable task patterns with lazy instantiation |
| TaskConfig decorators | Ad-hoc behavior composition without interfaces |
| AffinityScheduler | Per-resource concurrency limits |
| AI Tool Registry | Tasks as AI agent tools with schema |

All patterns follow the ctor-closure principle: closures capture computation, Scope binds at invocation, behavior is declarative.

---

## Design Critique & Future Direction

> **Status:** These are critical notes on the patterns above. Several have fundamental issues that need resolution before implementation.

### Problem 1: Global State via FiberLocal::current()

The `FiberLocal::current()` pattern introduces ambient global state:

```php
<?php

// This is global state access -- problematic
$ctx = FiberLocal::current();
$url = $ctx?->get('url');
```

**Issues:**
- Implicit dependency -- code silently depends on fiber context existing
- Testing complexity -- must set up fiber context before testing
- Refactoring hazard -- can't grep for dependencies
- Violates explicit data flow principle

**Alternative approach:** Keep Scope threading explicit. If deep access is needed, consider a `ScopeAccessor` service registered in the scope that can be resolved where needed. The Scope IS the context -- don't create a parallel context system.

### Problem 2: Magic Strings Everywhere

Multiple patterns use magic strings for identification:

```php
<?php

// Magic string: 'http.get' -- what is this? Where's it defined?
TaskTemplate::define(name: 'http.get', ...);
TaskTemplate::get('http.get');

// Magic string: 'url' -- what type? Required? Optional?
FiberLocal::current()?->get('url');

// Magic string: 'search_database' -- typo-prone
$registry->register(name: 'search_database', ...);
```

**Issues:**
- No IDE autocomplete
- Typos cause runtime failures
- No type information
- Can't use "find all references"

**Alternative approach:** Use FQCNs or typed enums as identifiers:

```php
<?php

// FQCN as key -- typed, discoverable, refactorable
TaskTemplate::define(HttpGet::class, ...);
TaskTemplate::get(HttpGet::class);

// For templates that don't warrant a class, use marker interfaces or enums:
enum CommonTask: string {
    case HttpGet = 'http_get';
    case DbQuery = 'db_query';
}

TaskTemplate::define(CommonTask::HttpGet, ...);
```

### Problem 3: Rename 'work' to 'computation'

Throughout these patterns, the closure property is named `work`:

```php
<?php

final class ConfigurableTask implements Dispatchable
{
    public function __construct(
        private readonly Closure $work,  // Should be $computation
    ) {}
}
```

**Rationale:** "Computation" better captures the ctor-closure pattern's intent -- it's a suspended computation awaiting its environment (Scope). "Work" is generic; "computation" is precise.

### Problem 4: Unified Input Layer Pattern

The `ConfigurableTask` and `ToolRegistry` hint at something more powerful: a **unified input layer** where HTTP routes, console commands, AI tools, and MCP servers all use the same declaration pattern.

**Current fragmentation (typical frameworks):**
- HTTP: Route annotations, router files, controller classes
- Console: Command classes, console kernel registration
- AI Tools: Custom registry, schema definitions
- MCP: Another registration system

**Convoy opportunity:** One pattern for all input layers.

```php
<?php

declare(strict_types=1);

namespace Convoy\Input;

use Closure;

/**
 * Unified input handler -- works for HTTP, CLI, AI tools, MCP, etc.
 */
final readonly class Handler implements Dispatchable
{
    public function __construct(
        public Closure $computation,
        public HandlerConfig $config = new HandlerConfig(),
    ) {}

    public function __invoke(Scope $scope): mixed
    {
        return ($this->computation)($scope);
    }
}

/**
 * A group of handlers keyed by their input identifier (path, command name, tool name)
 */
final readonly class HandlerGroup implements Dispatchable
{
    /**
     * @param Closure(Scope): array<string, Handler> $factory
     */
    public function __construct(
        private Closure $factory,
        public GroupConfig $config = new GroupConfig(),
    ) {}

    public function __invoke(Scope $scope): array
    {
        return ($this->factory)($scope);
    }
}
```

**HTTP Routes (routes/api.php):**

```php
<?php
// routes/api.php

declare(strict_types=1);

use Convoy\Http\Route;
use Convoy\Http\RouteGroup;
use Convoy\Http\RouteConfig;
use Convoy\Scope;

return static fn(Scope $scope): RouteGroup => new RouteGroup([
    'GET /users' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->all(),
        config: new RouteConfig(timeout: 5.0),
    ),
    'GET /users/{id}' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->find(
            $s->attribute('route.id')
        ),
    ),
    'POST /users' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->create(
            $s->attribute('request.body')
        ),
        config: new RouteConfig(middleware: [AuthMiddleware::class]),
    ),
]);
```

**Console Commands (commands/db.php):**

```php
<?php
// commands/db.php

declare(strict_types=1);

use Convoy\Console\Command;
use Convoy\Console\CommandGroup;
use Convoy\Console\CommandConfig;
use Convoy\Scope;

return static fn(Scope $scope): CommandGroup => CommandGroup::of([
    'migrate' => new Command(
        fn: static fn(Scope $s) => $s->service(Migrator::class)->run(),
        config: static fn(CommandConfig $c) => $c
            ->withDescription('Run database migrations'),
    ),
    'db:seed' => new Command(
        fn: static fn(Scope $s) => $s->service(Seeder::class)->run(),
        config: static fn(CommandConfig $c) => $c
            ->withDescription('Seed the database'),
    ),
]);
```

**AI Tools (tools/database.php):**

```php
<?php
// tools/database.php

declare(strict_types=1);

use Convoy\AI\Tool;
use Convoy\AI\ToolGroup;
use Convoy\AI\ToolConfig;
use Convoy\AI\ToolParam;
use Convoy\Scope;

return static fn(Scope $scope): ToolGroup => new ToolGroup([
    'search_database' => new Tool(
        static fn(Scope $s) => $s->service(Database::class)->search(
            $s->attribute('tool.table'),
            $s->attribute('tool.query'),
        ),
        config: new ToolConfig(
            description: 'Search a database table',
            parameters: [
                'table' => ToolParam::string('Table name')->required(),
                'query' => ToolParam::string('Search query')->required(),
                'limit' => ToolParam::int('Max results')->default(10),
            ],
        ),
    ),
    'send_email' => new Tool(
        static fn(Scope $s) => $s->service(Mailer::class)->send(
            $s->attribute('tool.to'),
            $s->attribute('tool.subject'),
            $s->attribute('tool.body'),
        ),
        config: new ToolConfig(
            description: 'Send an email',
            parameters: [
                'to' => ToolParam::string('Recipient email')->required(),
                'subject' => ToolParam::string('Email subject')->required(),
                'body' => ToolParam::string('Email body')->required(),
            ],
        ),
    ),
]);

**Key insight:** The array key IS the input identifier (route path, command name, tool name). The value IS the handler with its computation and config. Same pattern, different input layers.

### Problem 5: ToolRegistry Type Safety

The current `ToolRegistry` accepts `Dispatchable` but should accept a typed `Tool` interface:

```php
<?php

// Current: loses type information
$registry->register(name: 'search', tool: new Task(...), schema: [...]);

// Better: typed Tool with schema as part of the type
interface Tool extends Dispatchable
{
    public ToolConfig $config { get; }
}

final readonly class AiTool implements Tool
{
    public function __construct(
        public Closure $computation,
        public ToolConfig $config,
    ) {}

    public function __invoke(Scope $scope): mixed
    {
        return ($this->computation)($scope);
    }
}
```

**File-based tool definition (tools/support.php):**

```php
<?php
// tools/support.php

declare(strict_types=1);

use Convoy\AI\Tool;
use Convoy\AI\ToolGroup;
use Convoy\AI\ToolConfig;
use Convoy\Scope;

return static fn(Scope $scope): ToolGroup => new ToolGroup([
    'search_database' => new Tool(
        static fn(Scope $s) => $s->service(Database::class)->search(...),
        config: new ToolConfig(description: '...', parameters: [...]),
    ),
    'send_email' => new Tool(
        static fn(Scope $s) => $s->service(Mailer::class)->send(...),
        config: new ToolConfig(description: '...', parameters: [...]),
    ),
]);

// ToolGroup is itself Dispatchable -- returns the array of tools
// The agent receives this and can iterate/dispatch
```

### Problem 6: AgentLoop as Convoy's Showcase

The `AgentLoop` pattern represents the intersection of:
- Modern async PHP (fibers, concurrent execution)
- AI agent architecture (tool use, message threading)
- Convoy's strengths (Scope, behavior composition, cancellation)

**Current issues:**
- Uses FiberLocal for argument passing (global state)
- Magic strings for tool identification
- Weak typing on tool registration

**Reimagined AgentLoop:**

```php
<?php

declare(strict_types=1);

namespace Convoy\AI;

use Convoy\Scope;
use Convoy\Task\Dispatchable;

final class AgentLoop implements Dispatchable
{
    public function __construct(
        private readonly ToolGroup $tools,
        private readonly AgentConfig $config,
    ) {}

    public function __invoke(Scope $scope): AgentResult
    {
        $client = $scope->service($this->config->clientClass);
        $messages = $this->config->systemPrompt
            ? [['role' => 'system', 'content' => $this->config->systemPrompt]]
            : [];

        // Initial user message comes via scope attribute
        $messages[] = ['role' => 'user', 'content' => $scope->attribute('agent.input')];

        // Resolve tools once at loop start
        $availableTools = $scope->execute($this->tools);

        while (true) {
            $scope->throwIfCancelled();

            $response = $client->chat([
                'model' => $this->config->model,
                'messages' => $messages,
                'tools' => $this->buildSchemas($availableTools),
            ]);

            $choice = $response->choices[0];

            if ($choice->finishReason === 'stop') {
                return new AgentResult(
                    content: $choice->message->content,
                    messages: $messages,
                    toolCalls: [], // Could track all tool calls made
                );
            }

            if ($choice->finishReason === 'tool_calls') {
                $messages[] = $choice->message->toArray();

                // Execute tool calls concurrently
                // Arguments passed via scope attributes, not FiberLocal
                $results = $scope->concurrent(
                    array_map(
                        fn($call) => $this->wrapToolCall($availableTools, $call),
                        $choice->message->toolCalls,
                    ),
                );

                foreach ($choice->message->toolCalls as $i => $call) {
                    $messages[] = [
                        'role' => 'tool',
                        'tool_call_id' => $call->id,
                        'content' => json_encode($results[$i]),
                    ];
                }
            }
        }
    }

    private function wrapToolCall(array $tools, object $call): Dispatchable
    {
        $toolName = $call->function->name;
        $args = json_decode($call->function->arguments, true);

        if (!isset($tools[$toolName])) {
            throw new UnknownToolException($toolName);
        }

        // Return a task that sets args as attributes and executes the tool
        return new Task(static function(Scope $s) use ($tools, $toolName, $args): mixed {
            // Each arg becomes a scoped attribute: tool.{param_name}
            $toolScope = $s;
            foreach ($args as $key => $value) {
                $toolScope = $toolScope->withAttribute("tool.{$key}", $value);
            }
            return $toolScope->execute($tools[$toolName]);
        });
    }

    private function buildSchemas(array $tools): array
    {
        return array_map(
            static fn(Tool $tool): array => $tool->config->toOpenAISchema(),
            $tools,
        );
    }
}
```

**Agent definition (agents/support.php):**

```php
<?php
// agents/support.php

declare(strict_types=1);

use Convoy\AI\AgentLoop;
use Convoy\AI\AgentConfig;
use Convoy\AI\ToolGroup;
use Convoy\Scope;

return static fn(Scope $scope): AgentLoop => new AgentLoop(
    tools: ToolGroup::merge(
        (require __DIR__ . '/../tools/database.php')($scope),
        (require __DIR__ . '/../tools/email.php')($scope),
        (require __DIR__ . '/../tools/tickets.php')($scope),
    ),
    config: new AgentConfig(
        model: 'gpt-4',
        clientClass: OpenAIClient::class,
        systemPrompt: 'You are a helpful assistant.',
        maxIterations: 10,
        timeout: 300.0,
    ),
);
```

**Running the agent:**

```php
<?php
// bin/support-agent

$agent = (require __DIR__ . '/../agents/support.php')($scope);

$result = $scope
    ->withAttribute('agent.input', 'Find all overdue invoices and email a summary to billing@example.com')
    ->execute($agent);
```

**What makes this shine:**
- No global state -- arguments flow through Scope attributes
- No magic strings -- tool names are array keys, typed at definition
- Concurrent tool execution -- Convoy's `concurrent()` handles fiber coordination
- Cancellation support -- `throwIfCancelled()` respects scope cancellation
- Behavior composition -- AgentLoop itself can be wrapped with timeout, retry
- Testable -- mock the ToolGroup, verify attribute-based invocation

---

## Open Questions

1. **HandlerGroup vs dedicated types:** Should `RouteGroup`, `CommandGroup`, `ToolGroup` be separate classes or one generic `HandlerGroup<T>`?

2. **Schema generation:** For AI tools, should schema be defined declaratively (like above) or derived from the computation's reflection?

3. **Attribute namespacing:** Using `tool.{param}`, `route.{segment}`, `request.{field}` -- is this the right convention?

4. **Cross-input-layer middleware:** Can the same middleware work for HTTP auth, CLI auth, and tool auth?

5. **Discovery:** How do input layers discover their handlers? Boot-time registration? Attribute scanning? Service tagging?

---

## 6. File-Based Handler Discovery (rector.php Pattern)

**Problem:** Handler registration is scattered across application bootstrap, service providers, and configuration files. No single place to see what handlers exist for a given input layer.

**Solution:** PHP files return `Dispatchable` groups. Discovery is filesystem-based. Similar to rector.php returning a callable.

### Directory Structure

```
app/
├── routes/
│   ├── api.php      → returns RouteGroup
│   └── web.php      → returns RouteGroup
├── commands/
│   ├── migrate.php  → returns CommandGroup
│   └── deploy.php   → returns CommandGroup
├── tools/
│   ├── database.php → returns ToolGroup
│   └── email.php    → returns ToolGroup
└── agents/
    ├── support.php  → returns AgentLoop
    └── analyst.php  → returns AgentLoop
```

### File Contracts

Each file returns a closure that receives `Scope` and returns a `Dispatchable`:

```php
<?php
// routes/api.php

declare(strict_types=1);

use Convoy\Http\Route;
use Convoy\Http\RouteGroup;
use Convoy\Scope;

return static fn(Scope $scope): RouteGroup => new RouteGroup([
    'GET /users' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->all(),
    ),
    'GET /users/{id}' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->find(
            $s->attribute('route.id')
        ),
    ),
    'POST /users' => new Route(
        static fn(Scope $s) => $s->service(UserRepository::class)->create(
            $s->attribute('request.body')
        ),
    ),
]);
```

```php
<?php
// tools/database.php

declare(strict_types=1);

use Convoy\AI\Tool;
use Convoy\AI\ToolGroup;
use Convoy\AI\ToolConfig;
use Convoy\AI\ToolParam;
use Convoy\Scope;

return static fn(Scope $scope): ToolGroup => new ToolGroup([
    'search_database' => new Tool(
        static fn(Scope $s) => $s->service(Database::class)->search(
            $s->attribute('tool.table'),
            $s->attribute('tool.query'),
            $s->attribute('tool.limit') ?? 10,
        ),
        config: new ToolConfig(
            description: 'Search a database table for matching records',
            parameters: [
                'table' => ToolParam::string('Table name to search')->required(),
                'query' => ToolParam::string('Search query')->required(),
                'limit' => ToolParam::int('Max results')->default(10),
            ],
        ),
    ),
    'insert_record' => new Tool(
        static fn(Scope $s) => $s->service(Database::class)->insert(
            $s->attribute('tool.table'),
            $s->attribute('tool.data'),
        ),
        config: new ToolConfig(
            description: 'Insert a record into a database table',
            parameters: [
                'table' => ToolParam::string('Table name')->required(),
                'data' => ToolParam::object('Record data')->required(),
            ],
        ),
    ),
]);
```

```php
<?php
// agents/analyst.php

declare(strict_types=1);

use Convoy\AI\AgentLoop;
use Convoy\AI\AgentConfig;
use Convoy\AI\ToolGroup;
use Convoy\Scope;

return static fn(Scope $scope): AgentLoop => new AgentLoop(
    tools: ToolGroup::merge(
        (require __DIR__ . '/../tools/database.php')($scope),
        (require __DIR__ . '/../tools/charts.php')($scope),
    ),
    config: new AgentConfig(
        model: 'claude-sonnet-4-20250514',
        clientClass: AnthropicClient::class,
        systemPrompt: <<<'PROMPT'
            You are a data analyst. You can search databases, generate
            visualizations, and compile reports for stakeholders.
            PROMPT,
        maxIterations: 20,
        timeout: 300.0,
    ),
);
```

### Discovery & Loading

```php
<?php

declare(strict_types=1);

namespace Convoy\Discovery;

use Convoy\Scope;
use Convoy\Task\Dispatchable;
use Closure;

final class HandlerLoader
{
    /** @var array<string, Closure(Scope): Dispatchable> */
    private array $handlers = [];

    public function __construct(
        private readonly string $basePath,
    ) {}

    /**
     * Discover all handler files in a directory
     */
    public function discover(string $directory): self
    {
        $path = $this->basePath . '/' . $directory;

        if (!is_dir($path)) {
            return $this;
        }

        foreach (glob($path . '/*.php') as $file) {
            $key = $directory . '/' . basename($file, '.php');
            $this->handlers[$key] = require $file;
        }

        return $this;
    }

    /**
     * Load a specific handler
     */
    public function load(string $key, Scope $scope): Dispatchable
    {
        if (!isset($this->handlers[$key])) {
            throw new \InvalidArgumentException("Unknown handler: {$key}");
        }

        return ($this->handlers[$key])($scope);
    }

    /**
     * Load all handlers in a directory
     *
     * @return array<string, Dispatchable>
     */
    public function loadAll(string $directory, Scope $scope): array
    {
        $result = [];

        foreach ($this->handlers as $key => $factory) {
            if (str_starts_with($key, $directory . '/')) {
                $name = substr($key, strlen($directory) + 1);
                $result[$name] = $factory($scope);
            }
        }

        return $result;
    }

    /**
     * Get all registered handler keys
     */
    public function keys(): array
    {
        return array_keys($this->handlers);
    }
}
```

### Application Integration

```php
<?php
// bin/serve

use Convoy\Application;
use Convoy\Discovery\HandlerLoader;
use Convoy\Runner\HttpRunner;

$app = Application::starting($config)
    ->providers(new AppBundle())
    ->compile();

$loader = (new HandlerLoader(__DIR__ . '/../app'))
    ->discover('routes');

$runner = new HttpRunner($app, $loader);
$runner->run('0.0.0.0:8080');
```

```php
<?php
// bin/agent

use Convoy\Application;
use Convoy\Discovery\HandlerLoader;

$app = Application::starting($config)
    ->providers(new AppBundle())
    ->compile();

$app->startup();
$scope = $app->createScope();

$loader = (new HandlerLoader(__DIR__ . '/../app'))
    ->discover('agents');

$agent = $loader->load('agents/analyst', $scope);

$result = $scope
    ->withAttribute('agent.input', $argv[1] ?? 'Analyze Q4 sales')
    ->execute($agent);

echo $result->content . PHP_EOL;

$scope->dispose();
$app->shutdown();
```

### Composition Patterns

**Merging tool groups:**

```php
<?php
// tools/all.php -- aggregates all tool groups

return static fn(Scope $scope): ToolGroup => ToolGroup::merge(
    (require __DIR__ . '/database.php')($scope),
    (require __DIR__ . '/email.php')($scope),
    (require __DIR__ . '/calendar.php')($scope),
);
```

**Conditional loading:**

```php
<?php
// tools/admin.php -- only available to admins

return static fn(Scope $scope): ToolGroup => new ToolGroup(
    $scope->attribute('user.isAdmin')
        ? [
            'delete_user' => new Tool(...),
            'reset_password' => new Tool(...),
        ]
        : []
);
```

**Lazy loading via ToolGroup factory:**

```php
<?php

final readonly class ToolGroup implements Dispatchable
{
    /**
     * @param Closure(Scope): array<string, Tool>|array<string, Tool> $tools
     */
    public function __construct(
        private Closure|array $tools,
    ) {}

    public function __invoke(Scope $scope): array
    {
        return is_array($this->tools)
            ? $this->tools
            : ($this->tools)($scope);
    }

    public static function merge(self ...$groups): self
    {
        return new self(static fn(Scope $s): array => array_merge(
            ...array_map(fn($g) => $g($s), $groups)
        ));
    }
}
```

**Key insight:** Files are the registration mechanism. The filesystem IS the registry. No annotations, no service provider boilerplate. Just PHP files that return typed Dispatchables.

---

## 7. Priority-Based Middleware Ordering via Attributes

**Problem:** Middleware registration order determines execution order, but this is fragile. Moving a line of code changes behavior. There's no declarative way to express "this middleware runs before that one."

**Solution:** PHP 8 attributes declare ordering intent. Compiler resolves to sorted array at boot time.

### Attribute Definitions

```php
<?php
// middleware/attributes.php

declare(strict_types=1);

namespace Convoy\Middleware\Attribute;

use Attribute;

/**
 * Numeric priority. Higher values run first (outermost in onion).
 * Default is 0. Negative values run after default.
 */
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Priority
{
    public function __construct(
        public int $value,
    ) {}
}

/**
 * This middleware must run before the specified middleware.
 * Creates a dependency edge: $this -> $target
 */
#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
final readonly class Before
{
    /** @param class-string $middleware */
    public function __construct(
        public string $middleware,
    ) {}
}

/**
 * This middleware must run after the specified middleware.
 * Creates a dependency edge: $target -> $this
 */
#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
final readonly class After
{
    /** @param class-string $middleware */
    public function __construct(
        public string $middleware,
    ) {}
}

/**
 * Middleware only applies to handlers in specified groups.
 * Empty groups array means applies to all.
 */
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Group
{
    /** @param string|list<string> $groups */
    public function __construct(
        public string|array $groups,
    ) {
        $this->groups = is_array($groups) ? $groups : [$groups];
    }
}

/**
 * Named stage for coarse-grained ordering.
 * Stages run in order: 'early' -> 'auth' -> 'main' -> 'late'
 */
#[Attribute(Attribute::TARGET_CLASS)]
final readonly class Stage
{
    public const EARLY = 'early';
    public const AUTH = 'auth';
    public const MAIN = 'main';
    public const LATE = 'late';

    public function __construct(
        public string $stage = self::MAIN,
    ) {}
}
```

### Middleware Examples

```php
<?php
// middleware/RateLimitMiddleware.php

declare(strict_types=1);

namespace App\Middleware;

use Convoy\Middleware\Attribute\Priority;
use Convoy\Middleware\Attribute\Stage;
use Convoy\Middleware\Attribute\Group;
use Convoy\Middleware\TaskMiddleware;
use Convoy\Scope;
use Convoy\Task\Dispatchable;

#[Priority(100)]
#[Stage(Stage::EARLY)]
#[Group('api')]
final class RateLimitMiddleware implements TaskMiddleware
{
    public function process(Dispatchable $task, Scope $scope, callable $next): mixed
    {
        $key = $scope->attribute('request.ip');
        $limiter = $scope->service(RateLimiter::class);

        if (!$limiter->attempt($key)) {
            throw new TooManyRequestsException();
        }

        return $next();
    }
}
```

```php
<?php
// middleware/AuthMiddleware.php

#[Priority(90)]
#[Stage(Stage::AUTH)]
#[Group(['api', 'web'])]
final class AuthMiddleware implements TaskMiddleware
{
    public function process(Dispatchable $task, Scope $scope, callable $next): mixed
    {
        $token = $scope->attribute('request.auth_token');

        if ($token === null) {
            throw new UnauthorizedException();
        }

        $user = $scope->service(Auth::class)->validate($token);
        $scope = $scope->withAttribute('user', $user);

        return $next();
    }
}
```

```php
<?php
// middleware/ApiKeyMiddleware.php

#[Before(AuthMiddleware::class)]
#[Group('api')]
final class ApiKeyMiddleware implements TaskMiddleware
{
    public function process(Dispatchable $task, Scope $scope, callable $next): mixed
    {
        $apiKey = $scope->attribute('request.api_key');

        if ($apiKey !== null) {
            // API key auth bypasses normal auth
            $scope = $scope->withAttribute('request.auth_token',
                $scope->service(ApiKeyService::class)->tokenize($apiKey)
            );
        }

        return $next();
    }
}
```

```php
<?php
// middleware/MetricsMiddleware.php

#[Priority(-100)]
#[Stage(Stage::LATE)]
final class MetricsMiddleware implements TaskMiddleware
{
    public function process(Dispatchable $task, Scope $scope, callable $next): mixed
    {
        $start = hrtime(true);

        try {
            return $next();
        } finally {
            $elapsed = (hrtime(true) - $start) / 1e6;
            $scope->service(Metrics::class)->record('request.duration', $elapsed);
        }
    }
}
```

### Middleware Compiler

```php
<?php
// middleware/MiddlewareCompiler.php

declare(strict_types=1);

namespace Convoy\Middleware;

use Convoy\Middleware\Attribute\Priority;
use Convoy\Middleware\Attribute\Before;
use Convoy\Middleware\Attribute\After;
use Convoy\Middleware\Attribute\Group;
use Convoy\Middleware\Attribute\Stage;
use ReflectionClass;

final class MiddlewareCompiler
{
    private const STAGE_ORDER = [
        Stage::EARLY => 1000,
        Stage::AUTH => 500,
        Stage::MAIN => 0,
        Stage::LATE => -500,
    ];

    /**
     * @param list<class-string<TaskMiddleware>> $middlewareClasses
     * @return list<MiddlewareDefinition>
     */
    public function compile(array $middlewareClasses): array
    {
        $definitions = [];

        foreach ($middlewareClasses as $class) {
            $definitions[$class] = $this->extractDefinition($class);
        }

        return $this->sort($definitions);
    }

    /**
     * @param list<MiddlewareDefinition> $all
     * @return list<MiddlewareDefinition>
     */
    public function forGroup(array $all, string $group): array
    {
        return array_values(array_filter(
            $all,
            static fn(MiddlewareDefinition $def): bool =>
                $def->groups === [] || in_array($group, $def->groups, true)
        ));
    }

    private function extractDefinition(string $class): MiddlewareDefinition
    {
        $ref = new ReflectionClass($class);
        $attrs = $ref->getAttributes();

        $priority = 0;
        $stage = Stage::MAIN;
        $before = [];
        $after = [];
        $groups = [];

        foreach ($attrs as $attr) {
            $instance = $attr->newInstance();

            match ($instance::class) {
                Priority::class => $priority = $instance->value,
                Stage::class => $stage = $instance->stage,
                Before::class => $before[] = $instance->middleware,
                After::class => $after[] = $instance->middleware,
                Group::class => $groups = $instance->groups,
                default => null,
            };
        }

        // Combine stage and priority into effective priority
        $effectivePriority = self::STAGE_ORDER[$stage] + $priority;

        return new MiddlewareDefinition(
            class: $class,
            priority: $effectivePriority,
            before: $before,
            after: $after,
            groups: $groups,
        );
    }

    /**
     * Topological sort respecting Before/After edges, then by priority
     *
     * @param array<class-string, MiddlewareDefinition> $definitions
     * @return list<MiddlewareDefinition>
     */
    private function sort(array $definitions): array
    {
        // Build dependency graph
        $edges = []; // edges[A] = [B, C] means A must come before B and C

        foreach ($definitions as $class => $def) {
            $edges[$class] ??= [];

            // "Before X" means this runs before X, so X depends on this
            foreach ($def->before as $target) {
                if (isset($definitions[$target])) {
                    $edges[$class][] = $target;
                }
            }

            // "After X" means this depends on X
            foreach ($def->after as $target) {
                if (isset($definitions[$target])) {
                    $edges[$target] ??= [];
                    $edges[$target][] = $class;
                }
            }
        }

        // Kahn's algorithm for topological sort
        $inDegree = array_fill_keys(array_keys($definitions), 0);
        foreach ($edges as $deps) {
            foreach ($deps as $dep) {
                $inDegree[$dep]++;
            }
        }

        // Start with nodes that have no dependencies
        $queue = new \SplPriorityQueue();
        foreach ($inDegree as $class => $degree) {
            if ($degree === 0) {
                // Higher priority = process first
                $queue->insert($class, $definitions[$class]->priority);
            }
        }

        $sorted = [];
        while (!$queue->isEmpty()) {
            $class = $queue->extract();
            $sorted[] = $definitions[$class];

            foreach ($edges[$class] ?? [] as $dependent) {
                $inDegree[$dependent]--;
                if ($inDegree[$dependent] === 0) {
                    $queue->insert($dependent, $definitions[$dependent]->priority);
                }
            }
        }

        // Check for cycles
        if (count($sorted) !== count($definitions)) {
            $remaining = array_filter($inDegree, fn($d) => $d > 0);
            throw new CyclicMiddlewareDependencyException(array_keys($remaining));
        }

        return $sorted;
    }
}
```

### MiddlewareDefinition Value Object

```php
<?php

declare(strict_types=1);

namespace Convoy\Middleware;

final readonly class MiddlewareDefinition
{
    /**
     * @param class-string<TaskMiddleware> $class
     * @param list<class-string<TaskMiddleware>> $before
     * @param list<class-string<TaskMiddleware>> $after
     * @param list<string> $groups
     */
    public function __construct(
        public string $class,
        public int $priority,
        public array $before,
        public array $after,
        public array $groups,
    ) {}
}
```

### Integration with Application

```php
<?php
// middleware/discovery.php

declare(strict_types=1);

use Convoy\Middleware\MiddlewareCompiler;

return static fn(Scope $scope): array => (new MiddlewareCompiler())->compile([
    RateLimitMiddleware::class,
    AuthMiddleware::class,
    ApiKeyMiddleware::class,
    ValidationMiddleware::class,
    LoggingMiddleware::class,
    MetricsMiddleware::class,
]);
```

```php
<?php
// Application bootstrap

$compiler = new MiddlewareCompiler();
$allMiddleware = $compiler->compile($middlewareClasses);

// For API routes, filter by group
$apiMiddleware = $compiler->forGroup($allMiddleware, 'api');

// Resolved order for 'api' group:
// 1. RateLimitMiddleware   (priority 1100: EARLY + 100)
// 2. ApiKeyMiddleware      (before AuthMiddleware)
// 3. AuthMiddleware        (priority 590: AUTH + 90)
// 4. ValidationMiddleware  (priority 0: MAIN)
// 5. LoggingMiddleware     (priority -500: LATE)
// 6. MetricsMiddleware     (priority -600: LATE + -100)
```

### Execution Order Visual

```
Request → [Rate] → [ApiKey] → [Auth] → [Validation] → Handler
                                                         ↓
Response ← [Rate] ← [ApiKey] ← [Auth] ← [Validation] ← Result
                                                         ↓
         ← [Logging] ← [Metrics] ← ← ← ← ← ← ← ← ← ← ← ←
```

Higher priority middleware wraps lower priority. The onion:
- Outermost: RateLimitMiddleware (first in, last out)
- Innermost: Handler

### Benefits

1. **Declarative** - Intent expressed at class definition
2. **Greppable** - `grep -r "#\[Priority"` finds all priorities
3. **IDE support** - Attributes are typed, autocomplete works
4. **Compile-time validation** - Cycle detection before runtime
5. **Group scoping** - Different middleware stacks for different contexts

### Relation to Existing TaskMiddleware

This builds on the existing `TaskMiddleware` interface. The attributes add ordering metadata; the interface defines the contract:

```php
<?php

interface TaskMiddleware
{
    public function process(Dispatchable $task, Scope $scope, callable $next): mixed;
}
```

**Key insight:** Attributes declare *metadata* (priority, ordering, groups). Interfaces declare *behavior* (the process method). This follows the "interfaces for behavior, attributes for metadata" principle.

---

## Memory Monitor for Long-Running HTTP Servers

**Problem:** PHP's memory limit kills the process when exceeded, but provides no early warning. In long-running servers (ReactPHP, FrankenPHP worker mode), slow memory leaks accumulate silently until OOM termination.

**Design Goals:**
1. Zero overhead on the request hot path
2. Configurable thresholds and responses
3. Trend detection (not just absolute thresholds)
4. Integration with existing logging/metrics infrastructure
5. Optional forced GC when trends indicate leaks

### Proposed API

```php
<?php

declare(strict_types=1);

namespace Convoy\Diagnostics;

use Closure;
use React\EventLoop\Loop;
use React\EventLoop\TimerInterface;

/**
 * Monitors memory usage in long-running processes.
 *
 * Uses periodic sampling (not per-request) to minimize overhead.
 * Detects both absolute thresholds and growth trends.
 */
final class MemoryMonitor
{
    /** Percentage of memory_limit that triggers warning. */
    private const float DEFAULT_WARN_THRESHOLD = 0.75;

    /** Percentage of memory_limit that triggers critical alert. */
    private const float DEFAULT_CRITICAL_THRESHOLD = 0.90;

    /** Number of samples to keep for trend analysis. */
    private const int SAMPLE_WINDOW = 60;

    /** Minimum growth rate (bytes/second) to consider a leak. */
    private const int LEAK_THRESHOLD_BYTES_PER_SEC = 10_000;

    private ?TimerInterface $timer = null;
    private bool $running = false;

    /** @var list<array{timestamp: float, bytes: int}> */
    private array $samples = [];

    private int $memoryLimitBytes;

    public function __construct(
        private readonly float $intervalSeconds = 30.0,
        private readonly float $warnThreshold = self::DEFAULT_WARN_THRESHOLD,
        private readonly float $criticalThreshold = self::DEFAULT_CRITICAL_THRESHOLD,
        private readonly bool $forceGcOnLeak = false,
        private readonly ?Closure $onWarn = null,
        private readonly ?Closure $onCritical = null,
        private readonly ?Closure $onLeakDetected = null,
    ) {
        $this->memoryLimitBytes = $this->parseMemoryLimit();
    }

    /**
     * Create with sensible defaults for production.
     *
     * Logs warnings at 75%, critical at 90%, checks every 30s.
     */
    public static function production(?Closure $logger = null): self
    {
        $log = $logger ?? static fn(string $level, string $msg, array $ctx) => error_log("[$level] $msg");

        return new self(
            intervalSeconds: 30.0,
            onWarn: static fn(MemorySnapshot $snap) => $log(
                'warning',
                "Memory at {$snap->percentUsed}% ({$snap->usedMb}MB / {$snap->limitMb}MB)",
                $snap->toArray()
            ),
            onCritical: static fn(MemorySnapshot $snap) => $log(
                'critical',
                "Memory critical at {$snap->percentUsed}% ({$snap->usedMb}MB / {$snap->limitMb}MB)",
                $snap->toArray()
            ),
            onLeakDetected: static fn(LeakReport $report) => $log(
                'warning',
                "Possible memory leak: {$report->growthRateKbPerMin}KB/min over {$report->windowSeconds}s",
                $report->toArray()
            ),
        );
    }

    /**
     * Create with aggressive settings for development/debugging.
     *
     * Lower thresholds, faster sampling, forced GC on leak detection.
     */
    public static function development(?Closure $logger = null): self
    {
        $log = $logger ?? static fn(string $level, string $msg, array $ctx) => error_log("[$level] $msg");

        return new self(
            intervalSeconds: 5.0,
            warnThreshold: 0.50,
            criticalThreshold: 0.75,
            forceGcOnLeak: true,
            onWarn: static fn(MemorySnapshot $snap) => $log('warning', "Memory: {$snap->percentUsed}%", []),
            onCritical: static fn(MemorySnapshot $snap) => $log('critical', "Memory critical: {$snap->percentUsed}%", []),
            onLeakDetected: static fn(LeakReport $report) => $log('warning', "Leak detected: {$report->growthRateKbPerMin}KB/min", []),
        );
    }

    /** Start the periodic monitoring timer. */
    public function start(): void
    {
        if ($this->running) {
            return;
        }

        $this->running = true;
        $this->timer = Loop::addPeriodicTimer($this->intervalSeconds, $this->sample(...));

        // Take initial sample
        $this->sample();
    }

    /** Stop monitoring and cancel the timer. */
    public function stop(): void
    {
        if (!$this->running) {
            return;
        }

        $this->running = false;

        if ($this->timer !== null) {
            Loop::cancelTimer($this->timer);
            $this->timer = null;
        }
    }

    /** Get current memory snapshot without triggering callbacks. */
    public function snapshot(): MemorySnapshot
    {
        return new MemorySnapshot(
            usedBytes: memory_get_usage(true),
            peakBytes: memory_get_peak_usage(true),
            limitBytes: $this->memoryLimitBytes,
        );
    }

    /** Analyze trend from sample history. */
    public function analyzeTrend(): ?LeakReport
    {
        if (count($this->samples) < 3) {
            return null;
        }

        $first = $this->samples[0];
        $last = $this->samples[array_key_last($this->samples)];

        $durationSeconds = $last['timestamp'] - $first['timestamp'];
        if ($durationSeconds < 10) {
            return null; // Not enough data
        }

        $bytesGrown = $last['bytes'] - $first['bytes'];
        $growthRate = $bytesGrown / $durationSeconds;

        if ($growthRate <= self::LEAK_THRESHOLD_BYTES_PER_SEC) {
            return null; // Growth within normal bounds
        }

        return new LeakReport(
            startBytes: $first['bytes'],
            endBytes: $last['bytes'],
            windowSeconds: $durationSeconds,
            growthRatePerSecond: $growthRate,
            sampleCount: count($this->samples),
        );
    }

    /** Force a GC cycle and return bytes freed. */
    public function forceGc(): int
    {
        $before = memory_get_usage(true);
        gc_collect_cycles();
        $after = memory_get_usage(true);

        return max(0, $before - $after);
    }

    private function sample(): void
    {
        $snapshot = $this->snapshot();

        // Store sample for trend analysis
        $this->samples[] = [
            'timestamp' => microtime(true),
            'bytes' => $snapshot->usedBytes,
        ];

        // Keep window bounded
        if (count($this->samples) > self::SAMPLE_WINDOW) {
            array_shift($this->samples);
        }

        // Check absolute thresholds
        if ($snapshot->percentUsed >= $this->criticalThreshold * 100) {
            $this->onCritical?->__invoke($snapshot);
        } elseif ($snapshot->percentUsed >= $this->warnThreshold * 100) {
            $this->onWarn?->__invoke($snapshot);
        }

        // Check for leak trend
        $leak = $this->analyzeTrend();
        if ($leak !== null) {
            $this->onLeakDetected?->__invoke($leak);

            if ($this->forceGcOnLeak) {
                $this->forceGc();
            }
        }
    }

    private function parseMemoryLimit(): int
    {
        $limit = ini_get('memory_limit');

        if ($limit === '-1') {
            return PHP_INT_MAX; // Unlimited
        }

        $unit = strtolower(substr($limit, -1));
        $value = (int) $limit;

        return match ($unit) {
            'g' => $value * 1024 * 1024 * 1024,
            'm' => $value * 1024 * 1024,
            'k' => $value * 1024,
            default => $value,
        };
    }
}

/**
 * Point-in-time memory state.
 */
final readonly class MemorySnapshot
{
    public int $usedMb;
    public int $peakMb;
    public int $limitMb;
    public float $percentUsed;
    public float $percentPeak;

    public function __construct(
        public int $usedBytes,
        public int $peakBytes,
        public int $limitBytes,
    ) {
        $this->usedMb = (int) ($usedBytes / 1024 / 1024);
        $this->peakMb = (int) ($peakBytes / 1024 / 1024);
        $this->limitMb = (int) ($limitBytes / 1024 / 1024);
        $this->percentUsed = $limitBytes > 0 ? round(($usedBytes / $limitBytes) * 100, 1) : 0;
        $this->percentPeak = $limitBytes > 0 ? round(($peakBytes / $limitBytes) * 100, 1) : 0;
    }

    /** @return array<string, mixed> */
    public function toArray(): array
    {
        return [
            'used_bytes' => $this->usedBytes,
            'peak_bytes' => $this->peakBytes,
            'limit_bytes' => $this->limitBytes,
            'used_mb' => $this->usedMb,
            'peak_mb' => $this->peakMb,
            'limit_mb' => $this->limitMb,
            'percent_used' => $this->percentUsed,
            'percent_peak' => $this->percentPeak,
        ];
    }
}

/**
 * Memory leak analysis report.
 */
final readonly class LeakReport
{
    public float $growthRateKbPerMin;

    public function __construct(
        public int $startBytes,
        public int $endBytes,
        public float $windowSeconds,
        public float $growthRatePerSecond,
        public int $sampleCount,
    ) {
        $this->growthRateKbPerMin = round(($growthRatePerSecond * 60) / 1024, 2);
    }

    /** @return array<string, mixed> */
    public function toArray(): array
    {
        return [
            'start_bytes' => $this->startBytes,
            'end_bytes' => $this->endBytes,
            'window_seconds' => $this->windowSeconds,
            'growth_rate_bytes_per_sec' => $this->growthRatePerSecond,
            'growth_rate_kb_per_min' => $this->growthRateKbPerMin,
            'sample_count' => $this->sampleCount,
        ];
    }
}
```

### Integration with HttpRunner

```php
<?php

// In HttpRunner constructor or run()
$this->memoryMonitor = MemoryMonitor::production(
    logger: fn($level, $msg, $ctx) => $this->app->trace()->log(
        TraceType::Diagnostic,
        $msg,
        $ctx
    )
);

// In run() after setting up signal handlers
$this->memoryMonitor->start();

// In shutdown()
$this->memoryMonitor->stop();
```

### Configuration via ApplicationBuilder

```php
<?php

// Optional: Wire up via ApplicationBuilder
Application::starting($context)
    ->providers(new AppBundle())
    ->memoryMonitor(MemoryMonitor::production())
    ->compile();
```

### Key Design Decisions

1. **Periodic timer, not per-request** - Checking memory every request adds latency. A 30s timer has negligible overhead and catches trends.

2. **Trend detection over absolute thresholds** - A slow leak at 40% memory is more actionable than a spike to 80% that settles back down.

3. **Callbacks instead of exceptions** - Memory pressure shouldn't crash requests. Log, alert, maybe force GC, but keep serving.

4. **Optional forced GC** - Off by default in production (avoid latency), on in development (surface leaks faster).

5. **Sliding window** - Keep 60 samples max. Old data isn't useful for detecting current leaks.

6. **Factory methods** - `production()` and `development()` encode best practices so users don't misconfigure.

### Future Extensions

- **Prometheus/StatsD export** - Emit metrics for grafana dashboards
- **Request-correlated sampling** - Track memory delta per-request type
- **Heap dump trigger** - At critical threshold, dump heap for analysis
- **Graceful shutdown** - At critical, stop accepting new requests and drain
