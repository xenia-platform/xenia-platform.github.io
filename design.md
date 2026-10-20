# Design

How xenia is built: what it promises, and the model behind it. For using it, see the [configuration](configuration.md) and [running](running.md) guides.

## Core Design Principles

- Strict separation between data plane and control plane
- No scripting in request execution path
- Immutable route graph snapshots
- Atomic state transitions
- Lock-free hot path
- Command-driven configuration updates
- Arena-based snapshot memory management

## Architecture

### Data Plane (C + libuv)

Responsible for:
- HTTP request handling
- TLS termination
- routing and upstream selection
- serving files, read on libuv's threadpool so that no event loop waits for the disk
- connection lifecycle management

Properties:
- no runtime scripting
- no locks in request path
- operates only on immutable snapshots


### Control Plane (QuickJS)

Responsible for:
- configuration compilation
- administrative operations
- emitting control commands
- introspection and system management

Properties:
- does not handle requests
- produces typed commands only
- validates and orchestrates state changes

## State Model

All runtime configuration is represented as:

> immutable, versioned route graph snapshots

Each snapshot contains:
- routing rules
- upstream definitions
- TLS configuration
- feature flags
- precomputed routing indexes

Snapshots are never modified after creation.

## State Transitions

1. Current ACTIVE snapshot is read-only
2. Control plane produces a batch of commands
3. A new snapshot is constructed from the active state
4. Commands are applied during construction
5. New snapshot is validated
6. Atomic pointer swap publishes the new state
7. Old snapshot is retired asynchronously

This ensures zero downtime updates and consistent request behavior.

## Memory Model

Xenia uses arena-based allocation for snapshot construction:

- Each snapshot has its own memory arena
- All configuration objects are allocated within that arena
- Snapshots are discarded as a whole unit
- Old arenas are released after safe retirement

This avoids fragmentation and simplifies memory ownership.

## Concurrency Model

- Request path is lock-free, but for what routes opt into sharing between workers: [rate limits](configuration.md#rate-limits) and [response caches](configuration.md#caching), each a striped table locked for a lookup at a time
- State is immutable during request execution
- Besides the snapshot pointer and those tables, the hot path touches only its own worker's counters, access log buffer and small-file cache
- File I/O runs on libuv's threadpool, shared by the workers; a file response hands its bytes back to its worker's loop
- All mutations happen outside request processing

## Control Protocol

All configuration changes are expressed as typed commands:

- route updates
- upstream modifications
- TLS configuration changes
- feature flag toggles
- system introspection queries

Commands are validated in C before being applied to a new snapshot.

## Goals

- High-performance request processing in C
- Predictable and deterministic runtime behavior
- Safe and atomic configuration updates
- Minimal runtime complexity
- Clear separation of control and execution concerns

## Non-Goals

- Runtime scripting of request handling
- Per-request dynamic logic execution
- Plugin-based execution inside hot path
- Mutable shared state in request processing
