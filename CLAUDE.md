# CLAUDE.md

## Design Philosophy

### Intent Before Execution

Convoy separates *what you want* from *how it runs*. Tasks are pure descriptions of work. The execution layer handles coordination, concurrency, cancellation. A developer writes:

```php
<?php

$scope->concurrent([new FetchCustomer($id), new ValidateInventory($items)]);
```

The underlying machinery—fibers, event loop, scheduling—stays invisible.

### Invokables as Computations with Identity

Closures are anonymous. Invokable classes have identity—the class name IS the computation's name. They're traceable, testable, serializable. Stack traces show `FetchUser::__invoke`, not `Closure@handler.php:47`.

### The Scope Split

- **Scope**: Minimal interface—service resolution, attributes, tracing
- **ExecutionScope**: Full capabilities—concurrency, cancellation, disposal

Most code needs only Scope. Handlers opt into ExecutionScope when they need concurrency primitives.

## Closure Requirements

**Task::of() enforces static closures** via reflection. Non-static closures capture `$this`, creating reference cycles that leak memory in long-running processes.

```php
<?php

// GOOD - explicit capture, no hidden references
Task::of(static fn(ExecutionScope $es) => $es->service(Repo::class)->find($id))

// BAD - captures $this implicitly, memory leak risk
Task::of(fn() => $this->repo->find($id))
```

When you need object state, extract to local variables:

```php
<?php

$sym = $this->symbol;
Task::of(static fn(ExecutionScope $es) => $es->service(Reader::class)->read($sym))
```

## Commands

```bash
composer test                              # Run all tests
composer cs                                # Check code style
composer cs:fix                            # Fix code style
php vendor/bin/phpunit --filter testName   # Single test
```

## Architecture

### Scope Hierarchy

```
Scope (interface)
├── service(), attribute(), withAttribute(), trace()
│
ExecutionScope (interface extends Scope)
├── execute(), concurrent(), race(), any(), map()
├── series(), waterfall(), settle(), timeout(), retry()
├── delay(), defer(), throwIfCancelled(), cancellation()
├── onDispose(), dispose(), $isCancelled
│
ExecutionLifecycleScope (implementation)

Scopeable (interface) - SEPARATE
├── __invoke(Scope $scope): mixed

Executable (interface) - SEPARATE
├── __invoke(ExecutionScope $scope): mixed
```

**Scope**: Minimal interface—service resolution, attributes, tracing.
**ExecutionScope**: Full execution—concurrency primitives, cancellation, disposal.
**Scopeable**: Tasks needing only service resolution.
**Executable**: Tasks needing full execution capabilities.

The `execute()` method accepts `Scopeable|Executable` union type.

### Handler System

```
Route/Command         ->  invokables with fn + config
RouteGroup/CommandGroup  ->  typed collections ("GET /path" keys)
HandlerGroup          ->  internal dispatch mechanics
HandlerLoader         ->  file-based discovery
```

| Type | Location | Purpose |
|------|----------|---------|
| `Route` | `src/Http/Route.php` | HTTP handler invokable |
| `RouteGroup` | `src/Http/RouteGroup.php` | Typed route collection |
| `RouteConfig` | `src/Http/RouteConfig.php` | Route config (methods, pattern, middleware) |
| `Command` | `src/Console/Command.php` | CLI handler invokable |
| `CommandGroup` | `src/Console/CommandGroup.php` | Typed command collection |
| `CommandConfig` | `src/Console/CommandConfig.php` | Command config (description, args) |

### File Loading Pattern

```php
<?php

// routes/api.php - file receives Scope, returns RouteGroup
return static fn(Scope $s): RouteGroup => RouteGroup::of([
    'GET /users' => new Route(
        fn: static fn(ExecutionScope $es) => $es->service(UserRepo::class)->all(),
    ),
]);
```

File closures receive `Scope` (service resolution). Handler `fn` closures receive `ExecutionScope` (full execution).

### Task System

```php
<?php

// Quick task via factory (implements Scopeable)
Task::of(static fn(ExecutionScope $es) => $es->service(SomeService::class)->work());

// Invokable class - Scopeable for service resolution only
final class DoWork implements Scopeable {
    public function __invoke(Scope $scope): mixed {
        return $scope->service(SomeService::class)->work();
    }
}

// Invokable class - Executable for concurrency primitives
final class DoConcurrentWork implements Executable {
    public function __invoke(ExecutionScope $scope): mixed {
        return $scope->concurrent([...]);
    }
}
```

### Behavioral Interfaces

Tasks declare behavior through PHP 8.4 property hooks:

| Interface | Property | Purpose |
|-----------|----------|---------|
| `Retryable` | `RetryPolicy $retryPolicy { get; }` | Automatic retry with policy |
| `HasTimeout` | `float $timeout { get; }` | Automatic timeout in seconds |
| `HasPriority` | `int $priority { get; }` | Priority queue ordering |
| `UsesPool` | `UnitEnum $pool { get; }` | Pool-aware scheduling |
| `Traceable` | `string $traceName { get; }` | Custom trace label |

### Service System

```php
<?php

class AppBundle implements ServiceBundle {
    public function services(Services $services, array $context): void {
        $services->singleton(DbPool::class)
            ->factory(fn() => new DbPool($context['db_url']))
            ->onShutdown(fn($p) => $p->drain());

        $services->scoped(RequestLogger::class)
            ->lazy()
            ->onDispose(fn($log) => $log->flush());
    }
}
```

### Key Files

| File | Purpose |
|------|---------|
| `src/Scope.php` | Minimal scope interface |
| `src/ExecutionScope.php` | Full execution interface |
| `src/ExecutionLifecycleScope.php` | Concrete implementation |
| `src/Application.php` | App bootstrap, createScope() |
| `src/Handler/HandlerLoader.php` | load(), loadDirectory() - file-based handler discovery |
| `src/Runner/HttpRunner.php` | ReactPHP HTTP server |
| `src/Runner/ConsoleRunner.php` | CLI dispatch |
| `src/Task/Task.php` | Task::of() factory |
| `src/Concurrency/RetryPolicy.php` | Retry strategies |
| `src/Concurrency/CancellationToken.php` | Cancellation primitives |

## Code Conventions

### PHP 8.4+ Features

| Feature | Use Case |
|---------|----------|
| Property hooks (`get`/`set`) | Behavioral interfaces, computed state |
| `readonly` classes | Immutable value objects, tasks |
| Asymmetric visibility (`private(set)`) | Public read, restricted write |
| `ReflectionClass::newLazyGhost()` | Lazy service initialization |
| `array_find`, `array_any`, `array_all` | Short-circuit collection ops |

### Style Rules

- All PHP blocks require `<?php` opening tag
- Static closures only in Task::of()—enforced at runtime
- `protected(set)` over `readonly` for property hooks compatibility
- Formal docblocks (`/** */`) for design decisions only
- No inline comments unless explaining non-obvious invariants

### Immutability Pattern

Tasks and configs use immutable builders:

```php
<?php

public function with(TaskConfig $config): self {
    return new self($this->work, $config);  // New instance
}
```
