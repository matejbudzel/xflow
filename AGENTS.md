# XFlow Agent Guide

This file defines repository-wide rules for coding agents and contributors.

Read this file before making changes. Also read `README.md`, `ARCHITECTURE.md`, and `IMPLEMENTATION.md` when the task touches design or implementation sequencing.

## Document roles

Keep information in the right document.

- `README.md` contains only stable, high-level product concepts and fundamental principles that should remain useful long term.
- `ARCHITECTURE.md` describes the intended target architecture and how components relate, regardless of current implementation status.
- `IMPLEMENTATION.md` contains temporary implementation sequencing, open implementation questions, and scaffolding plans. It should eventually be deleted.
- `AGENTS.md` contains durable repository-wide contribution and agent rules.

Do not turn the README into an implementation log.

When a durable architectural decision changes, update `ARCHITECTURE.md` in the same change.

When temporary implementation assumptions become obsolete, remove or update them in `IMPLEMENTATION.md` rather than allowing contradictory documentation to accumulate.

## Language

Use **English only** in repository content, including:

- source code identifiers where naming is under our control,
- comments,
- docstrings,
- documentation,
- commit messages,
- user-facing strings unless localization is explicitly being implemented,
- examples and test descriptions.

## Source file size

Keep source files at or below **300 lines** whenever reasonably possible.

This limit applies to executable/source files, including Python, JavaScript/TypeScript, HTML with substantial logic, and similar implementation files. It does not apply to Markdown documentation, generated files, lockfiles, or data fixtures where splitting would reduce clarity.

Do not evade the limit by compressing readable code onto fewer lines. Split by responsibility before a file becomes a large multi-purpose module.

If a source file must exceed 300 lines for a strong technical reason, document the reason in the change and keep the exception narrow.

## Python-first tooling

Host and development utilities should be written in **Python** unless there is a compelling reason otherwise.

They should run on at least:

- macOS,
- Debian/Linux.

Prefer the Python standard library where practical. Add dependencies only when they materially simplify or improve the implementation.

Platform-specific behavior, such as native desktop notifications, should be isolated behind small adapters so the rest of the tool remains portable.

## CircuitPython compatibility

Code intended to run on the XTEink X4 must remain compatible with CircuitPython and the device's memory constraints.

Do not assume normal CPython features or packages are available on-device.

Keep platform-specific imports out of portable core modules.

Prefer small modules, explicit data flow, limited allocations, and simple data formats over framework-heavy abstractions.

## Dependency direction

Preserve the architecture boundary:

- core logic must not import CircuitPython hardware modules,
- core logic must not import simulator/web-server code,
- core logic must not import host utilities,
- transports should call shared application services rather than duplicate domain behavior,
- platform adapters implement narrow capabilities consumed by the application.

If a proposed change requires the core to know whether it is on real hardware or in the simulator, first look for a missing abstraction.

## Hardware and simulator parity

New platform-dependent behavior should be exposed through an abstraction that can be represented in the desktop simulator when useful.

The simulator is not a mock rewrite of the product. It should run the same core and UI logic with fake adapters.

When adding hardware capabilities such as Wi-Fi, BLE, serial, battery state, storage, or time, consider whether the simulator needs a controllable fake for that capability.

## Storage

Mutable persistent data should use the storage abstraction.

On the device, prefer the SD card for:

- configuration,
- task data,
- downloaded digest content,
- runtime state,
- caches,
- staged updates.

Do not spread hard-coded `/sd/...` paths through application logic.

The application and bootstrap must handle unavailable or broken SD storage without entering an unrecoverable crash loop.

## Power

Treat battery life as a functional requirement.

- Do not leave Wi-Fi enabled by default.
- Do not leave BLE enabled by default.
- Prefer short, explicit radio sessions.
- Do not introduce polling loops that keep the MCU or radios active without a clear need.
- Activity timing must not require network connectivity.

## Time

Keep monotonic time and wall-clock time conceptually separate.

- Monotonic time drives activity timers, inactivity timeouts, and durations.
- Wall-clock time is optional and may be unknown.
- Core functionality must not fail because real date/time has not yet been synchronized.

## Rendering

Application UI targets a logical 800×480 grayscale surface.

Do not couple screen composition directly to the physical e-paper driver.

Keep e-paper refresh costs in mind. Avoid designs that require continuous or second-by-second full-screen updates.

## Transports

USB serial and HTTP are transport adapters around shared services.

Do not implement separate task semantics, settings behavior, or update rules independently in each transport.

BLE is primarily an optional short-lived event output unless the architecture is deliberately changed.

## Data formats

Prefer formats that are:

- human-readable when practical,
- cheap to parse on CircuitPython,
- deterministic,
- easy to generate from Python host tools.

Do not introduce JSON or a complex schema automatically when a simple line-oriented format is sufficient.

## Error handling and recovery

Prefer recoverable failure modes.

Examples:

- failed digest download keeps the previous good digest,
- failed storage access exposes a diagnostic state,
- failed application update must not unnecessarily require reflashing CircuitPython,
- unavailable wall-clock time degrades the UI rather than breaking task timing,
- unavailable network leaves offline features usable.

## Tests

Write tests primarily against portable CPython-compatible logic.

Important areas include:

- state machines,
- timing semantics,
- task parsing,
- storage behavior,
- service behavior,
- error and recovery paths,
- simulator/platform contract behavior.

Avoid tests that require physical hardware unless the behavior cannot be meaningfully tested otherwise.

## Scope discipline

Prefer small coherent changes.

Do not add speculative frameworks, generalized plugin systems, or abstraction layers that are not justified by a current architectural need.

When implementation exposes a flaw in the documented architecture, change the architecture explicitly rather than silently working around it.

## Repository hygiene

- Never commit credentials, Wi-Fi passwords, authentication-bearing private URLs, or other secrets.
- Provide example configuration with clearly fake values when examples are needed.
- Keep generated runtime data, simulator state, caches, downloaded digests, and local credentials out of Git.
- Keep commit messages concise and in English.
