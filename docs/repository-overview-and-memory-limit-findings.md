# Repository Overview and `--memory-limit` Findings

## What this repository is

`gtopt` is a high-performance optimization codebase for **Generation and Transmission Expansion Planning (GTEP)**.
Core solver logic is in modern C++ and models power-system operations/investments as LP/MIP problems.

## High-level codebase structure

- `include/gtopt/`: public C++ API headers (domain model, LP builders, solver interfaces, options, pools).
- `source/`: C++ implementations for LP assembly, planning methods, execution paths, and runtime infrastructure.
- `standalone/`: CLI binary entrypoint (`gtopt` executable).
- `plugins/`: solver backend plugins (OSI/CLP/CBC, CPLEX, HiGHS).
- `test/source/`: C++ unit tests (doctest).
- `cases/`: sample planning cases (JSON + time-series data).
- `scripts/`: Python package with converters/validators/utilities.
- `webservice/`: Next.js/TypeScript web service.
- `guiservice/`: Flask/Python GUI service.
- `docs/`: user/developer documentation.

## Key technologies used

- **C++26** + **CMake** for core solver/library.
- **Apache Arrow/Parquet** for high-volume time-series I/O.
- **DAW JSON Link** for JSON deserialization into C++ structs.
- **spdlog** for logging.
- **Boost.Container (`flat_map`)** for sparse-structure performance.
- **doctest** for C++ unit tests.
- **Python** tooling in `scripts/` and `guiservice/`.
- **Next.js + TypeScript** in `webservice/`.

## How code is organized conceptually

1. **Domain model**: C++ structs represent buses, lines, generators, storage, hydro, time/scenario hierarchy, and top-level planning objects.
2. **LP assembly layer**: `*LP` classes convert domain entities into LP variables/constraints and sparse matrix structures.
3. **Solver abstraction**: plugin-discovered backend interface allows swapping solver engines without changing core model assembly.
4. **Planning methods**: monolithic, SDDP, and cascade implementations orchestrate solve strategy.
5. **I/O layer**: JSON in; Parquet/CSV out; structured output per component.

## `--memory-limit` behavior tracing

### 1) Parsing and option mapping

- CLI accepts forms like `4096`, `300M`, `5G`, `1.5GB`.
- Parsing is done by `parse_memory_size()` and converted to MB.
- CLI option resolution writes this value into `planning.options.sddp_options.pool_memory_limit_mb`.
- `--memory-quota` (percentage of MemTotal) overrides `--memory-limit` when both are set.

### 2) Where the value is used

- Planning options are propagated into SDDP runtime options (`pool_memory_limit_mb`).
- Work-pool factories pass that into pool config as `max_process_rss_mb` (RSS throttle threshold), including SDDP and write-out pools.

### 3) What happens when memory limit is extremely low (e.g., ~100 MB)

This flag is implemented as **work-pool throttling**, not a hard OS memory cap:

- The scheduler admits tasks conservatively based on memory pressure gates (memory %, free memory, process RSS projection, swap pressure).
- With a configured RSS limit, admission is rate-limited and checked against projected RSS (`current_rss + measured_per_task`).
- **Critical safety rule:** when no task is active, the pool always admits one task to avoid deadlock/livelock.

So with a very low limit (like 100 MB), especially if process baseline RSS is already near/above that:

- The pool still makes progress (because one task is always admitted when idle).
- Concurrency can collapse toward near-serial execution.
- Throughput drops sharply; runs can become much slower.
- Throttle counters/warnings are expected.
- This does **not** guarantee RSS stays under 100 MB at all times; it throttles new dispatches to control growth.

### 4) Direct test evidence in repo

`test_work_pool.cpp` contains explicit deadlock-regression tests:

- A case with absurdly low RSS limit (`1 MB`, below resident floor) verifies all tasks still complete.
- Another case with impossible memory-percent gate (`0%`) also verifies completion via idle-progress guarantee.

These tests confirm the intended behavior under extreme limits: **slow/serialized but non-deadlocking progress**.

## Practical conclusion for `--memory-limit=100M`

Expected behavior in real runs:

- The solve should continue, but likely with heavy throttling.
- Effective parallelism may be very low.
- Runtime can increase significantly.
- If baseline process RSS already exceeds 100 MB, the limit functions mostly as a “do not increase concurrency” throttle rather than a strict cap.
