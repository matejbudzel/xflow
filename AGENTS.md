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

Build/release helpers, simulator launchers, test wrappers, serial tools, BLE listeners, and update-manifest generators all fall under this Python-first rule.

## CircuitPython compatibility

Code intended to run on the XTEink X4 must remain compatible with CircuitPython and the device's memory constraints.

Do not assume normal CPython features or packages are available on-device.

Keep platform-specific imports out of portable core modules.

Prefer small modules, explicit data flow, limited allocations, streaming I/O, and simple data formats over framework-heavy abstractions.

Where practical, portable/device-intended Python should pass a smoke test under CircuitPython's Unix port in addition to normal CPython tests. The Unix port is a compatibility check, not hardware emulation and not a substitute for real X4 tests.

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

The simulator should mirror the physical X4's application-visible button layout: four bottom buttons, two side buttons, and Power.

Battery simulation must provide at least full, medium, low, and critical presets plus an independently controllable USB/external-power state.

## Storage

Mutable persistent data should use the storage abstraction.

On the device, prefer the SD card for:

- configuration,
- task data,
- downloaded digest content,
- runtime state,
- caches,
- staged updates.

The baseline configuration location is `/sd/system/configuration.json`, but application/core code must not spread hard-coded `/sd/...` paths throughout the codebase.

The application and bootstrap must handle unavailable or broken SD storage without entering an unrecoverable crash loop.

Large document inputs such as XTH must be treated as streamable/SD-backed data rather than assumed to fit in RAM.

## Power

Treat battery life as a functional requirement.

- Do not leave Wi-Fi enabled by default.
- Do not leave BLE enabled by default.
- Prefer short, explicit radio sessions.
- Do not introduce polling loops that keep the MCU or radios active without a clear need.
- Activity timing must not require network connectivity.
- Treat Power as the required deep-sleep wake control; do not add wake support for the other six UI buttons without evidence that the complexity is worthwhile.

## Time

Keep monotonic time and wall-clock time conceptually separate.

- Monotonic time drives activity timers, inactivity timeouts, and durations.
- Wall-clock time is optional and may be unknown.
- Core functionality must not fail because real date/time has not yet been synchronized.

## Rendering

Application UI targets a logical **800×480 1-bit black/white framebuffer**.

Do not couple screen composition directly to the physical e-paper driver.

If source content is grayscale or 2-bit, convert/dither it before it reaches the shared logical framebuffer. The browser simulator and physical device must consume the same final 1-bit pixels rather than render different-quality versions of the UI.

Keep the global e-paper refresh policy separate from individual screens. The baseline is partial/fast refresh with a forced full refresh after 15 partial presentations, configurable at system level.

Keep e-paper refresh costs in mind. Avoid designs that require continuous or second-by-second full-screen updates.

## Transports

USB serial and HTTP are transport adapters around shared services.

Do not implement separate task semantics, settings behavior, or update rules independently in each transport.

BLE is primarily an optional short-lived event output unless the architecture is deliberately changed.

HTTP management uses REST-style JSON services. The on-device management SPA is a static build artifact served by the device and should not contain a second implementation of application behavior.

## Updates and build artifacts

Application updates are manifest-driven.

Build tooling should produce a deterministic release/build identifier, a file list, and integrity metadata. Candidate files are staged and verified before promotion.

Keep `code.py` outside normal application replacement unless a deliberate bootstrap migration is being implemented.

Do not make the device depend on a particular development web server. nginx may host artifacts during development, but the protocol contract is ordinary HTTP(S).

## Data formats

Prefer formats that are:

- human-readable when practical,
- cheap to parse on CircuitPython,
- deterministic,
- easy to generate from Python host tools.

Use JSON for structured configuration and REST API payloads. Do not introduce JSON or a complex schema automatically for line-oriented data such as the simple task list when a simpler format is sufficient.

## Error handling and recovery

Prefer recoverable failure modes.

Examples:

- failed digest download keeps the previous good digest,
- failed storage access exposes a diagnostic state,
- failed application update must not unnecessarily require reflashing CircuitPython,
- unavailable wall-clock time degrades the UI rather than breaking task timing,
- unavailable network leaves offline features usable.

## Tests

Use a three-level confidence model where practical:

1. **CPython tests** for fast unit/integration coverage of portable logic.
2. **CircuitPython Unix-port smoke tests** for runtime/language compatibility of portable/device-intended modules that do not require real board bindings.
3. **Real XTEink X4 tests** for display, buttons, sleep/wake, SD, battery, USB, Wi-Fi, BLE, and other hardware behavior.

Important automated areas include:

- state machines,
- timing semantics,
- task parsing,
- storage behavior,
- service behavior,
- error and recovery paths,
- simulator/platform contract behavior,
- deterministic 1-bit rendering,
- global partial/full refresh-cycle behavior.

Avoid tests that require physical hardware when the behavior can be meaningfully verified at a lower layer.

## Scope discipline

Prefer small coherent changes.

Do not add speculative frameworks, generalized plugin systems, or abstraction layers that are not justified by a current architectural need.

When implementation exposes a flaw in the documented architecture, change the architecture explicitly rather than silently working around it.

## Repository hygiene

- Never commit credentials, Wi-Fi passwords, authentication-bearing private URLs, or other secrets.
- Provide example configuration with clearly fake values when examples are needed.
- Keep generated runtime data, simulator state, caches, downloaded digests, staged update artifacts, and local credentials out of Git unless a specific fixture is intentionally committed.
- Keep commit messages concise and in English.
