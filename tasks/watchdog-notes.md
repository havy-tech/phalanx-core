# Monorepo Migration Watchdog Notes

## Baseline
- Commit: 3740ef8 (before monorepo refactor)
- Tests: 274 / 573 (expected), currently 274 / 516 with 42 ERRORS
- Date: 2026-03-15
- Packages at start: convoy-core, convoy-stream

## Issues Found

### [20:05] Phase 2 - Iteration 0 (Bootstrap)

**Changes observed (unstaged in working tree):**
- `src/ExecutionScope.php` -- stream() REMOVED, now extends `Scope, StreamContext` (DONE)
- `src/ExecutionLifecycleScope.php` -- MODIFIED (stream() removal expected)
- `src/Service/DeferredScope.php` -- stream() REMOVED (DONE)
- `src/Support/ExecutionScopeDelegate.php` -- stream() REMOVED (DONE)
- `src/Task/LazySequence.php` -- fromStream() REMOVED, dead imports cleaned (DONE)
- `src/Task/Collect.php` -- updated to `Convoy\Stream\Contract\StreamSource` (DONE)
- `src/Task/Drain.php` -- updated to `Convoy\Stream\Contract\StreamSource` (DONE)
- `src/Task/First.php` -- updated (assumed same)
- `src/Task/Reduce.php` -- updated (assumed same)
- `src/Stream/Contract/` -- 4 abstract files created with new namespace (DONE)
- Old concrete stream files (Channel, Emitter, ScopedStream, StreamEvent, Terminal/*) DELETED from core
- `examples/` directory DELETED
- `composer.json` MODIFIED

**convoy-stream package:**
- `src/` has: Channel.php, Emitter.php, ScopedStream.php, StreamEvent.php, Terminal/
- `ScopedStream::from()` factory EXISTS (DONE)
- composer.json has correct deps (core, react/async, react/event-loop, react/stream, evenement)
- Tests partially moved: Unit/{ChannelTest, EmitterTest, StreamEventTest} present
- Integration/ and Support/ dirs exist but empty integration tests

**Phase 2 Checklist:**
- [x] `stream()` removed from `ExecutionScope.php` interface
- [x] `stream()` removed from `DeferredScope.php`
- [x] `stream()` removed from `ExecutionScopeDelegate.php`
- [ ] `stream()` removed from `ExecutionLifecycleScope.php` -- needs verification
- [x] `LazySequence::fromStream()` removed
- [x] Dead imports cleaned from LazySequence (Channel, ReadableStreamInterface)
- [x] 4 abstract files in `src/Stream/Contract/` with new namespace
- [x] Namespace imports updated (Collect, Drain, LazySequence use Contract\*)
- [x] 7 concrete files moved to convoy-stream
- [x] `ScopedStream::from()` static factory created
- [x] convoy-stream composer.json correct
- [ ] convoy-core composer.json: react/stream NOT yet removed (not in require, only suggest - OK)
- [ ] convoy-core composer.json: evenement NOT in require (OK - was never there or already removed)
- [x] convoy-core composer.json: react/async and react/promise KEPT

**BLOCKING Issues:**
1. **42 test errors** -- Test files not yet split/moved:
   - `tests/Unit/Stream/ChannelTest.php` -- still in core, references `Convoy\Stream\Channel` (moved to convoy-stream)
   - `tests/Unit/Stream/EmitterTest.php` -- still in core, same issue
   - `tests/Unit/Stream/StreamEventTest.php` -- still in core, same issue
   - `tests/Integration/Stream/LazySequenceStreamTest.php` -- still calls `fromStream()` (method removed)
   - `tests/Integration/Stream/EmitterStreamTest.php` -- still in core, references moved classes
   - Note: convoy-stream has copies of ChannelTest, EmitterTest, StreamEventTest in tests/Unit/ but core copies NOT deleted

2. **LazySequenceStreamTest needs splitting** -- fromStream tests should move to convoy-stream, remaining LazySequence tests stay in core

**WARNING Issues:**
3. `react/child-process` still in convoy-core require (Phase 5 concern, not blocking yet)
4. `bin/convoy-worker` still in convoy-core (Phase 5 concern)
5. `clue/docker-react` still in require-dev (Phase 3 concern - should move with console examples)
6. `react/http` still in require-dev (Phase 4 concern)

**CRITICAL architectural concern (TRAP #8):**
- Handler system (Handler.php, HandlerLoader.php, HandlerGroup.php) has hard imports from Console/Http namespaces
- When Console/Http types move to their own packages, these imports will break
- Parent plan does NOT address this. Will surface in Phase 3/4.

**Status:** NEEDS ATTENTION -- 42 test errors must be resolved before migration can proceed

### [20:11] Phase 2 - Iteration 1

**Changes observed:**
- No new commits (still on 3740ef8)
- Working tree same set of changes as Iteration 0
- New deletions in working tree since bootstrap:
  - `tests/Unit/Stream/ChannelTest.php` DELETED from core
  - `tests/Unit/Stream/EmitterTest.php` DELETED from core
  - `tests/Unit/Stream/StreamEventTest.php` DELETED from core (note: still shows in `tests/Integration/Stream/`)
  - `tests/Integration/Stream/EmitterStreamTest.php` DELETED from core
  - `tests/Integration/Stream/LazySequenceStreamTest.php` MODIFIED (fromStream tests removed, rest kept)
- Packages: still convoy-core, convoy-stream only

**Test results:**
- convoy-core: **232 tests, 516 assertions -- ALL PASSING**
- convoy-stream: cannot test (no vendor/ directory yet)

**Phase 2 Checklist update:**
- [x] Test files split: Unit stream tests moved to convoy-stream, deleted from core
- [x] EmitterStreamTest moved to convoy-stream integration tests
- [x] LazySequenceStreamTest split: fromStream tests -> `LazySequenceFromStreamTest.php` in convoy-stream
- [x] Remaining LazySequenceStreamTest stays in core (modified)
- [ ] convoy-stream tests not yet runnable (no composer install)

**Test count math:**
- Core: 274 -> 232 (42 tests removed)
- convoy-stream should have: ~42 tests (3 unit files + 2 integration files)
- Combined target: 274 -- CANNOT VERIFY until convoy-stream vendor is installed

**Resolved from Iteration 0:**
- BLOCKING issue #1 (42 test errors) RESOLVED -- tests now pass in core
- BLOCKING issue #2 (LazySequenceStreamTest split) RESOLVED

**Remaining issues:**
- WARNING: convoy-stream has no vendor/, tests unverifiable
- WARNING: react/child-process, bin/convoy-worker still in core (Phase 5)
- WARNING: clue/docker-react in require-dev (Phase 3)
- WARNING: react/http in require-dev (Phase 4)
- CRITICAL: Handler system circular deps with Console/Http (Phase 3/4)

**Status:** OK (core green) / WARNING (stream unverified)

### [20:14] Phase 2 - Iteration 2

**Changes observed:** None. Same commit (3740ef8), same working tree, same test results.
- convoy-core: 232 tests, 516 assertions -- ALL PASSING
- convoy-stream: still no vendor/
- Packages: convoy-core, convoy-stream (no new packages)

**Status:** IDLE -- parent session may be working on convoy-stream setup or planning next phase

### [20:17] Phase 3+4 - Iteration 3

**Changes observed:**
- No new commits (still 3740ef8)
- MAJOR working tree changes: Console AND Http files deleted from core simultaneously
- New packages appeared: `convoy-console`, `convoy-http`
- New file in core: `src/Handler/HandlerConfig.php` (base class for route/command config)
- Console tests deleted from core: ArgvParserTest, CommandValueObjectsTest, HelpGeneratorTest
- Http tests deleted from core: RouteConfigTest

**Deleted from core (Phase 3 - Console):**
- `src/Console/*` (13 files) -> convoy-console
- `src/Runner/ConsoleRunner.php` -> convoy-console

**Deleted from core (Phase 4 - Http):**
- `src/Http/*` (7 files) -> convoy-http
- `src/Runner/HttpRunner.php` -> convoy-http
- `src/Runtime/ConvoyHttpRunner.php` -> convoy-http

**Test results:**
- convoy-core: **186 tests, 389 assertions, 19 ERRORS**
- All 19 errors are Handler tests (HandlerGroupTest: 11, HandlerLoaderTest: 1, HandlerDispatchTest: 7)
- Root cause: `Class "Convoy\Console\CommandConfig" not found` and `Class "Convoy\Http\RouteConfig" not found`

**TRAP #8 CONFIRMED -- Handler circular dependency:**
- Parent created `HandlerConfig` base class (option b from plan) -- good approach
- BUT Handler.php, HandlerGroup.php, HandlerLoader.php STILL import Console/Http types directly:
  ```
  Handler.php:         use Convoy\Console\CommandConfig, Convoy\Http\RouteConfig
  HandlerGroup.php:    use Convoy\Console\{ArgvParser, CommandConfig, CommandScopeDecorator}
                       use Convoy\Http\{HttpScopeDecorator, QueryParams, RouteConfig, RouteParams}
  HandlerLoader.php:   use Convoy\Console\CommandGroup, Convoy\Http\RouteGroup
  ```
- These imports break because the classes moved to convoy-console/convoy-http but core doesn't depend on those packages
- Parent is mid-flight on this -- `@migration` annotations present, HandlerConfig base class created
- Resolution in progress but NOT yet complete

**Phase 3 Checklist:**
- [x] Console files moved to convoy-console
- [x] ConsoleRunner moved
- [ ] Handler imports from Console NOT yet resolved (BLOCKING)
- [ ] Console tests moved -- partially (some deleted from core, need to verify in convoy-console)
- [ ] CommandGroup readonly->private(set) -- unknown

**Phase 4 Checklist:**
- [x] Http files moved to convoy-http
- [x] HttpRunner and ConvoyHttpRunner moved
- [ ] Handler imports from Http NOT yet resolved (BLOCKING)
- [ ] RouteGroup readonly->private(set) -- unknown
- [ ] react/http removed from core deps -- unknown (composer.json modified but not checked)

**Status:** NEEDS ATTENTION -- 19 Handler test errors from circular dep (TRAP #8). Parent is actively working on it.

### [20:20] Phase 3+4 - Iteration 4

**Changes observed:** None since Iteration 3.
- Same commit (3740ef8), same working tree, same 19 Handler errors
- convoy-core: 186 tests, 389 assertions, 19 ERRORS
- Handler file mod times: Handler.php (Mar 8), HandlerConfig.php (Mar 9), HandlerGroup.php (Mar 15 16:03), HandlerLoader.php (Mar 15 16:02)
- Handler files NOT being actively modified right now -- parent may be working in convoy-console/convoy-http packages or replanning the Handler refactor

**Status:** STALLED on Handler circular dep. Waiting for parent to resolve TRAP #8.

### [20:23] Phase 3+4 - Iteration 5

**Changes observed in core:** None (same commit, same working tree, same 19 errors).
- convoy-core: 186 tests, 389 assertions, 19 ERRORS

**Activity in extracted packages:**
- Parent actively working in convoy-console and convoy-http (28 files newer than HandlerGroup.php)
- `CommandConfig` now `use Convoy\Handler\HandlerConfig` (correct direction: console -> core)
- `RouteConfig` now `use Convoy\Handler\HandlerConfig` (correct direction: http -> core)
- Dependency graph is partially correct but core still has reverse imports

**Handler refactor status:**
- HandlerConfig base class in core: DONE
- CommandConfig/RouteConfig extend HandlerConfig: DONE (in their packages)
- Core Handler/* still imports Console/Http types: NOT RESOLVED
- Expected next step: Either (a) Handler/* moves to the packages, or (b) core Handler/* refactored to use only HandlerConfig interface with type checks/protocol dispatch

**Status:** IN PROGRESS -- parent building out extracted packages, Handler circular dep unresolved

### [20:26] Phase 3+4 - Iteration 6

**Changes observed:** None. No PHP files modified anywhere in packages/ since last iteration.
- convoy-core: 186 tests, 389 assertions, 19 ERRORS (unchanged)
- No new packages, no new commits
- Parent session idle or context-switching

**Status:** IDLE -- same 19 Handler errors, no file activity

### [20:32] Phase 3+4 - Iteration 7

**Changes observed:** Root phpunit.xml modified (Smoke suite added then removed per transcript).
No PHP file changes. Core unchanged: 186 tests, 389 assertions, 19 ERRORS.

**Parent session status:** Investigating 17 missing tests (274 - 257 = 17).
- Breakdown: 186 core + 42 stream + 7 console + 22 http = 257 from root `-c phpunit.xml`
- Parent confused about git availability -- suggested they use `git diff 3740ef8 --name-status`
- Suggested option (a) for Handler deps: convoy/console + convoy/http as require-dev
- Parent appears to be reading suggestions / replanning

**Open issues (prioritized):**
1. BLOCKING: 19 Handler test errors in core standalone (circular Console/Http imports)
2. WARNING: 17 missing tests (not yet located -- need git diff to find)
3. WARNING: No new packages yet (convoy-parallel not started -- Phase 5)
4. INFO: Root phpunit.xml config collision with core's phpunit.xml

**Status:** WAITING -- parent digesting suggestions, no file activity

### [20:35] Phase 3+4+5 - Iteration 8

**Changes observed -- MAJOR PROGRESS:**

1. **`src/WorkerDispatch.php` CREATED** -- Phase 5 interface in core!
   - Namespace: `Convoy\WorkerDispatch` (correct)
   - Single method: `inWorker(Scopeable|Executable $task, ExecutionScope $scope): mixed` (matches plan)
   - Clean, minimal interface

2. **`composer.json` updated** -- Parent took option (a) for Handler deps:
   - `convoy/console` added to require-dev
   - `convoy/http` added to require-dev
   - `react/http` REMOVED from require-dev (Phase 4 cleanup)
   - `react/child-process` still in require (Phase 5 -- should move to convoy/parallel)
   - `bin/convoy-worker` still in core (Phase 5 -- should move)
   - `clue/docker-react` still in require-dev (should move to convoy-console)

3. **Missing tests being recovered:**
   - `convoy-console/tests/Unit/HelpGeneratorTest.php` CREATED
   - `convoy-console/tests/Unit/ArgvParserTest.php` CREATED
   - These were 2 of the 17 missing tests identified earlier

**Phase 5 Checklist (started):**
- [x] `WorkerDispatch.php` interface created in core root namespace
- [x] Interface has correct single method signature
- [ ] `ExecutionLifecycleScope::inWorker()` reduced to delegation -- not yet verified
- [ ] Worker methods removed from ExecutionLifecycleScope -- not yet verified
- [ ] `$supervisor` and `$parallelConfig` removed -- not yet verified
- [ ] `ParallelBundle` created -- not yet checked
- [ ] react/child-process removed from core -- NOT YET
- [ ] bin/convoy-worker moved -- NOT YET
- [ ] convoy-parallel package created -- NOT YET

**Core tests:** Still 186/389 with 19 errors (expected until `composer update` pulls in console/http as dev deps)

**Status:** ACTIVE PROGRESS -- Phase 5 started, missing tests being recovered, Handler circular dep addressed via require-dev

### [20:38] Phase 5 - Iteration 9

**Changes observed:**
- `src/Application.php` MODIFIED -- Phase 5 bundle registration
- `src/ApplicationBuilder.php` MODIFIED -- Phase 5 bundle registration
- `src/ExecutionLifecycleScope.php` MODIFIED -- inWorker() delegation refactored
- `phpunit.xml.dist` CREATED (addressing config collision suggestion)
- Additional test deletions now visible: `tests/Unit/Http/QueryParamsTest.php`, `tests/Unit/Http/RouteParamsTest.php`

**Test results: 186 tests, 382 assertions, 23 ERRORS, 1 FAILURE**
- 19 errors: Handler tests (same -- console/http not in autoloader standalone)
- 4 NEW errors: `InWorkerTest` -- worker tests can't find parallel classes (expected during Phase 5)
  - `InWorkerTest::executes_cpu_intensive_task`
  - `InWorkerTest::multiple_tasks_execute_sequentially`
  - `InWorkerTest::executes_simple_task_in_worker`
  - `InWorkerTest::parallel_tasks_execute_concurrently`
- 1 NEW failure: `InWorkerTest::propagates_exceptions_from_worker`
  - Expected: 'Intentional failure'
  - Got: 'Worker execution requires convoy/parallel. Install it via: composer require convoy/parallel'
  - This means `inWorker()` now throws a helpful error when convoy/parallel is not installed -- correct Phase 5 behavior!

**Phase 5 progress:**
- [x] WorkerDispatch interface created
- [x] ExecutionLifecycleScope::inWorker() refactored to throw helpful error without convoy/parallel
- [x] Application/ApplicationBuilder modified for bundle registration
- [ ] InWorkerTest needs to move to convoy-parallel (tests worker functionality that requires parallel package)
- [ ] react/child-process removal from core -- not yet
- [ ] bin/convoy-worker move -- not yet
- [ ] convoy-parallel package creation -- not yet visible

**Analysis:** The InWorker tests SHOULD move to convoy-parallel since they test worker execution. Keeping them in core will always fail since the parallel runtime won't be available. The `propagates_exceptions_from_worker` test failure confirms the delegation pattern is working correctly -- core now gives an actionable error instead of silently failing.

**Status:** ACTIVE PROGRESS -- Phase 5 ExecutionLifecycleScope refactor underway

### [20:41] Phase 5 - Iteration 10

**Changes observed -- ALL 5 PACKAGES NOW EXIST:**
- `convoy-parallel` package CREATED with 28 PHP files
- `bin/convoy-worker` moved to convoy-parallel
- convoy-parallel composer.json has correct deps (core, react/child-process, react/async, react/event-loop, react/promise)
- Src structure: Agent/, Dispatch/, Process/, Protocol/, Runtime/, Supervisor/, ParallelConfig.php

**Phase 5 Checklist update:**
- [x] WorkerDispatch interface in core
- [x] ExecutionLifecycleScope::inWorker() throws helpful error without convoy/parallel
- [x] Application/ApplicationBuilder modified for bundle registration
- [x] convoy-parallel package created with 28 files
- [x] bin/convoy-worker moved to convoy-parallel
- [x] convoy-parallel composer.json with correct deps
- [x] react/child-process in convoy-parallel (not core) -- CORRECT
- [ ] react/child-process REMOVED from core composer.json -- NOT YET (still in core require)
- [ ] InWorkerTest moved to convoy-parallel -- NOT YET (still failing in core)
- [ ] $supervisor/$parallelConfig removed from ExecutionLifecycleScope -- unverified
- [ ] ParallelBundle created -- need to check

**Core test results unchanged:** 186 tests, 382 assertions, 23 errors, 1 failure
- 19 Handler errors (need composer update with console/http dev deps)
- 4 InWorker errors + 1 failure (need tests moved to convoy-parallel)

**Remaining cleanup for core:**
1. Remove `react/child-process` from core composer.json require
2. Remove `bin/convoy-worker` from core composer.json bin
3. Move InWorkerTest to convoy-parallel
4. Run `composer update` in core with path repos to resolve console/http dev deps
5. Add `convoy/parallel` to core require-dev (for any remaining parallel integration tests)

**Status:** NEAR COMPLETION -- all 5 packages exist, cleanup phase

### [20:44] Phase 5 - Iteration 11

**Changes observed -- PHASE 5 NEARLY COMPLETE:**
- `bin/convoy-worker` DELETED from core (moved to convoy-parallel)
- `bin` section REMOVED from core composer.json
- ALL 25 `src/Parallel/*` files DELETED from core
- No `Convoy\Parallel\` imports remain in core src -- CLEAN separation
- InWorkerTest + parallel tests moved to convoy-parallel/tests/ (Unit + Integration)

**Test results: 166 tests, 337 assertions, 19 ERRORS**
- Down from 186/382/23+1 -- 20 tests moved to convoy-parallel
- Only 19 Handler errors remain (console/http require-dev autoloader issue)
- No InWorker failures -- those tests are in convoy-parallel now

**Phase 5 Checklist update:**
- [x] WorkerDispatch interface in core
- [x] ExecutionLifecycleScope::inWorker() refactored
- [x] Application/ApplicationBuilder modified
- [x] convoy-parallel package with 28 PHP files
- [x] bin/convoy-worker moved
- [x] bin section removed from core composer.json
- [x] All src/Parallel/* deleted from core
- [x] No Convoy\Parallel\ imports in core src
- [x] Tests moved to convoy-parallel (Unit + Integration)
- [ ] **react/child-process STILL IN core require** -- needs removal!

**Remaining cleanup (ONLY ITEM):**
1. Remove `react/child-process` from core composer.json require -- it's only needed by convoy-parallel now

**Overall migration status:**
- Phase 1 (monorepo infra): COMPLETE
- Phase 2 (stream extraction): COMPLETE
- Phase 3 (console extraction): COMPLETE (Handler deps via require-dev)
- Phase 4 (http extraction): COMPLETE (Handler deps via require-dev)
- Phase 5 (parallel extraction): 99% -- just react/child-process removal left

**Test count math:**
- Core: 166 (19 errors from Handler console/http deps)
- Stream: ~42
- Console: ~17 (after recovering missing tests)
- Http: ~22
- Parallel: ~20
- Expected total: ~267 (still 7 short of 274 -- missing tests not fully recovered)

**Status:** NEAR COMPLETE -- one dep removal from done

### [20:50] ALL PHASES COMPLETE - Iteration 12

**ROOT MONOREPO: 274 tests, 573 assertions -- ALL PASSING**

This matches the exact baseline: 274 tests, 573 assertions. Zero errors, zero failures.

**Changes since last iteration:**
- InWorkerTest fixed (was 4 errors + 1 failure, now all pass)
- react/child-process confirmed removed from core require
- convoy/parallel added to core require-dev
- All 5 packages fully operational from root

**Core standalone:** 166 tests, 337 assertions, 19 errors (Handler console/http deps -- expected without dev deps installed)

**Final package structure:**
- convoy-core: 166 tests (65 unit + 101 integration)
- convoy-stream: 42 tests (31 unit + 11 integration)
- convoy-console: 24 tests
- convoy-http: 22 tests
- convoy-parallel: 20 tests (15 unit + 5 integration)
- **Total: 274**

**Remaining cosmetic items:**
1. Core standalone 19 Handler errors (need `composer update` with path repos for dev deps)
2. `clue/docker-react` still in core require-dev (should be in convoy-console)

**Status:** MIGRATION COMPLETE -- root monorepo green at baseline invariant

## Final Remediation Checklist

(populated after migration completes)
