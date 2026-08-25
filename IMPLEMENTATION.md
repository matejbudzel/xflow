# XFlow Implementation Plan

This document tracks near-term implementation sequencing, unresolved implementation choices, and temporary scaffolding.

It is intentionally **not** a durable architecture document. Stable structural decisions belong in `ARCHITECTURE.md`. Once the implementation converges and the remaining work is better represented by issues/tests/code, this file should be deleted.

## Current status

The repository is at the initial design stage. The core architecture is documented, but the runtime, simulator, transports, storage layer, host utilities, build pipeline, and XTH viewer still need to be implemented.

## First implementation slice

The first useful vertical slice should prove that the same application logic can run on desktop Python and later on CircuitPython.

Target behavior:

```text
load task list
    -> show next task
    -> Start
    -> countdown using monotonic time
    -> Pause/Resume
    -> Stop or Complete
    -> advance to next task
```

The first slice does not need real XTEink hardware I/O.

### Suggested order

1. Define minimal core data structures and task state machine.
2. Define a small clock abstraction with monotonic time and optional wall-clock time.
3. Define storage abstraction and desktop filesystem implementation.
4. Define the shared 800×480 1-bit framebuffer/rendering abstraction.
5. Implement one task screen using the desktop bitmap backend.
6. Implement the seven synthetic X4 buttons in their physical layout.
7. Add the lightweight simulator HTTP server and browser controls.
8. Add tests around state transitions and timing behavior.
9. Add a CircuitPython Unix-port smoke-test path for portable/device-intended modules.

This proves the portable-core boundary before hardware-specific work starts.

## Simulator baseline

The simulator should become the default UI-development environment as early as possible.

Initial fake platform capabilities:

- exact 800×480 1-bit screen buffer,
- seven physical-button equivalents arranged as 2×2 bottom, 1×2 side, plus Power,
- monotonic clock,
- optional wall-clock time,
- persistent local storage directory,
- battery presets: full, medium, low, critical,
- independently switchable USB/external-power state.

Then add:

- optional exact battery voltage/percentage overrides,
- fake USB serial RX/TX,
- fake Wi-Fi off/client/AP state,
- fake Internet success/failure,
- fake BLE state and captured advertisements,
- missing/corrupt/read-only storage scenarios,
- simulated remote manifest/digest download results.

The browser should remain a thin control and inspection surface. Application behavior stays in Python.

## Rendering baseline

Use a single 1-bit rendering path from the start.

- UI code renders to the shared 800×480 black/white framebuffer.
- Source grayscale, including 2-bit XTH pages, is converted/dithered before reaching that framebuffer.
- The desktop bitmap renderer visualizes exactly those bits rather than using a richer browser-only representation.
- The physical adapter sends the same logical framebuffer to the X4 display.

Implement a global presentation/refresh controller independent from individual screens:

1. each committed presentation increments a partial-refresh counter,
2. normal presentations request partial/fast refresh,
3. after 15 partial presentations, force a full refresh,
4. reset the counter after a full refresh,
5. read the interval from configuration so hardware testing can tune it later.

The exact timer redraw cadence remains separate from this refresh-cycle policy.

## Device runtime baseline

After the desktop vertical slice is stable:

1. Add minimal `code.py` bootstrap.
2. Add `app.py` entry point.
3. Add XTEink display adapter.
4. Add XTEink button adapter for the four bottom buttons, two side buttons, and Power.
5. Mount and expose SD-backed storage.
6. Read battery voltage/USB state.
7. Verify monotonic timing behavior on hardware.
8. Verify Power-button wake from deep sleep.
9. Do not spend initial implementation effort on waking from the other six application buttons.
10. Verify partial/full refresh behavior and tune only if the default cycle proves unsuitable.

Hardware findings that change durable design assumptions must be reflected in `ARCHITECTURE.md`.

## Persistent data

Use the architecture baseline:

```text
/sd/
    system/
        configuration.json
        state.json
    tasks/
        current.txt
    digest/
        current.xth
    cache/
    updates/
```

For development, mirror the same logical structure under a configurable desktop directory such as `sim-data/`.

Storage paths below the storage abstraction may evolve without leaking into core/application code.

## Configuration

Start with JSON at `/sd/system/configuration.json` on the real device.

Initial fields should cover at least:

- Wi-Fi SSID/password,
- Wi-Fi behavior,
- digest URL,
- application update manifest URL,
- inactivity timeout,
- optional BLE-event behavior,
- full-refresh interval, defaulting to 15.

Use simple JSON types and avoid a configuration schema framework until there is evidence one is needed.

Do not put secrets into source files committed to Git.

## Runtime compatibility testing

Normal unit/integration tests run on CPython.

Add a second smoke-test command that runs portable/device-intended modules with a locally built CircuitPython Unix port when available. This layer is specifically intended to catch assumptions such as unsupported language/runtime behavior that normal CPython accepts.

The Unix port is not hardware emulation and does not provide every CircuitPython-only module. Tests that require board bindings must continue to use adapters/fakes or real hardware.

The desired testing ladder is:

```text
CPython tests
    -> CircuitPython Unix smoke
        -> XTEink X4 hardware smoke/integration tests
```

Do not block ordinary development on the native-port smoke test when the developer has not built the runtime locally; make the distinction visible in tooling/CI output instead.

## USB serial

Implement serial management before complex real-device network management because it provides a simple recovery/debug path.

Initial capabilities should include:

- set wall-clock Unix timestamp,
- push task list,
- get status,
- get battery/power status,
- emit task lifecycle events.

Start with a conventional line-oriented protocol. Keep request/response boundaries and asynchronous event messages explicit. The exact command vocabulary can evolve as the services take shape.

## HTTP management

Expose the shared management services as REST-style JSON endpoints.

Initial capabilities should include:

- status,
- task-list get/push,
- time setting,
- settings read/write,
- digest refresh/status,
- update check/apply.

The real device serves a static `index.html`/SPA on port 80 while HTTP management is active. The SPA is part of the application build and calls the REST API; it is not generated dynamically on-device.

The simulator should reuse the management SPA where practical and add simulator-only controls around it for fake hardware state.

Exact endpoint path naming is intentionally low-value and may evolve during implementation as long as transport/service boundaries stay clean.

## Wi-Fi

Implement explicit session-oriented networking rather than permanently connected Wi-Fi.

Client session baseline:

1. enable radio,
2. connect using stored credentials,
3. if connected, opportunistically attempt NTP sync,
4. if configured, refresh digest,
5. perform explicitly requested network operation,
6. disable Wi-Fi unless a management session is meant to stay active.

AP mode should provide local HTTP management without requiring upstream Internet access.

## BLE notifications

Treat BLE event broadcasting as an optional follow-up after core activity events and host tooling work.

Baseline:

```text
TASK_DONE
    -> enable BLE
    -> broadcast compact advertisement for a short interval
    -> disable BLE
```

The host listener should deduplicate events using an event sequence number or equivalent mechanism.

## Host tooling

All general host/dev utilities should be Python.

Initial useful commands may eventually form one CLI or remain small scripts. Required capabilities include:

- push tasks,
- send current Unix timestamp,
- query status/battery,
- listen to serial events,
- scan BLE advertisements,
- display a desktop notification,
- trigger digest refresh/update,
- launch simulator,
- build application artifacts/update manifest,
- run CPython tests,
- optionally run the CircuitPython Unix-port smoke suite.

Avoid premature packaging complexity. A small standard-library-first Python toolset is preferred initially.

## Daily digest

Digest support follows the initial task/timer vertical slice.

Before writing a new parser from assumptions, inspect CrossPoint Reader's XTH/XTCH parser and viewer behavior as the main XTEink-specific reference.

Implementation steps:

1. collect a few real daily-digest XTH samples and record their size/page characteristics,
2. understand the minimum header/page/index structures required from CrossPoint's implementation,
3. implement streaming/chunked access rather than loading the full file into RAM,
4. download the configured XTH URL into staged SD storage,
5. validate that the downloaded object is usable enough to replace the current copy,
6. atomically preserve/replace the last good digest,
7. decode the required 2-bit pages/content,
8. dither into the shared 1-bit framebuffer,
9. implement basic navigation/viewing,
10. add explicit refresh command,
11. hook opportunistic refresh into successful Wi-Fi client sessions.

XTH files may be multiple megabytes or larger. Treat SD as the working set and RAM as a small streaming/cache window.

Do not attempt complete generic XTH compatibility before the actual daily digest works.

## Application build and updates

The development machine produces a static application build that can be published by nginx or any equivalent ordinary HTTP server.

Start with a manifest such as:

```json
{
  "release": "20260825T154500Z",
  "files": [
    {"path": "app.py", "url": "app.py", "size": 1234, "sha256": "..."},
    {"path": "web/index.html", "url": "web/index.html", "size": 5678, "sha256": "..."}
  ]
}
```

The exact JSON field names may evolve, but the concepts are fixed: release/build ID, file list, locations, and integrity metadata.

Baseline flow:

1. user presses Check update in the management SPA or invokes the equivalent service,
2. device downloads the configured manifest,
3. compare release/build ID,
4. if newer, expose an explicit Update action,
5. download all candidate files to `/sd/updates/...`,
6. verify size/hash,
7. promote the staged build,
8. restart/reload,
9. let the stable bootstrap recover if the promoted application cannot start.

A timestamp-based release ID is sufficient initially if it is deterministic and sortable.

The main unresolved update detail is the exact promotion/rollback depth: whether to retain one complete previous application build, use per-file backups, or another minimal last-known-good strategy.

## Tests

Prioritize tests for code that can run under normal CPython:

- task parsing,
- task state transitions,
- timer pause/resume semantics,
- inactivity timeout behavior,
- optional wall-clock behavior,
- persistent state/storage failure handling,
- digest last-good-copy behavior,
- transport-independent service behavior,
- battery preset behavior,
- global partial/full refresh-cycle behavior,
- deterministic 1-bit rendering/dithering behavior.

Add CircuitPython Unix-port smoke coverage for portable/device-intended modules where the native port can execute them without real board bindings.

Rendering can additionally use deterministic image snapshots where useful, but tests should not depend exclusively on pixel snapshots.

## Remaining open implementation questions

The broad architecture is no longer intentionally open on most earlier questions. The remaining details should be resolved while implementing:

- exact action mapping and long/short-press semantics for the six normal UI buttons,
- timer-screen redraw cadence while a task is running,
- exact grayscale-to-1-bit dithering/threshold algorithm,
- exact serial command vocabulary/framing details,
- exact update promotion/rollback mechanism after staging,
- exact XTH subset/index/cache strategy required by real daily-digest files,
- exact native CircuitPython Unix-port build/run wrapper used by developer tooling and CI.

These are implementation details unless experiments expose a reason to change the durable architecture.

## Removal criteria

Delete this file when:

- the first-class components described in `ARCHITECTURE.md` have stable implementations,
- remaining work is represented by tests/issues/code comments rather than a broad implementation roadmap,
- this document no longer provides useful temporary sequencing information.
