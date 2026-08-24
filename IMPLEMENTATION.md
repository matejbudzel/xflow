# XFlow Implementation Plan

This document tracks near-term implementation sequencing, unresolved implementation choices, and temporary scaffolding.

It is intentionally **not** a durable architecture document. Stable structural decisions belong in `ARCHITECTURE.md`. Once the implementation converges and the remaining work is better represented by issues/tests/code, this file should be deleted.

## Current status

The repository is at the initial design stage. The core concepts are documented, but the runtime, simulator, transports, storage layer, and host utilities still need to be implemented.

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
4. Define a simple 800×480 rendering abstraction.
5. Implement one task screen using the desktop bitmap backend.
6. Implement synthetic button input.
7. Add the lightweight simulator HTTP server and browser controls.
8. Add tests around state transitions and timing behavior.

This proves the portable-core boundary before hardware-specific work starts.

## Simulator baseline

The simulator should become the default UI-development environment as early as possible.

Initial fake platform capabilities:

- 800×480 screen buffer,
- physical-button equivalents,
- monotonic clock,
- optional wall-clock time,
- persistent local storage directory,
- configurable battery/USB state.

Then add:

- fake USB serial RX/TX,
- fake Wi-Fi off/client/AP state,
- fake Internet success/failure,
- fake BLE state and captured advertisements,
- missing/corrupt/read-only storage scenarios.

The browser should remain a thin control and inspection surface. Application behavior stays in Python.

## Device runtime baseline

After the desktop vertical slice is stable:

1. Add minimal `code.py` bootstrap.
2. Add `app.py` entry point.
3. Add XTEink display adapter.
4. Add XTEink button adapter.
5. Mount and expose SD-backed storage.
6. Read battery voltage/USB state.
7. Verify monotonic timing behavior on hardware.
8. Verify deep-sleep/wake behavior and available wake buttons.

Hardware findings that change durable design assumptions must be reflected in `ARCHITECTURE.md`.

## Persistent data

The initial SD-backed layout may use:

```text
/sd/
    config/
    tasks/
    digest/
    state/
    cache/
    updates/
```

Do not over-specify filenames or formats until the corresponding feature is implemented.

For development, mirror the same logical structure under a configurable desktop directory such as `sim-data/`.

## Configuration

Configuration needs to cover at least:

- Wi-Fi SSID/password,
- Wi-Fi behavior,
- digest URL,
- application update source,
- inactivity timeout,
- optional BLE-event behavior.

The exact configuration serialization format is still open. Prefer a format that is:

- easy to inspect and edit,
- cheap to parse on CircuitPython,
- safe to update atomically enough for this device,
- easy to mirror in desktop tooling.

Do not put secrets into source files committed to Git.

## USB serial

Implement serial management before adding complex network management because it provides a simple recovery/debug path.

Initial capabilities should include:

- set wall-clock Unix timestamp,
- push task list,
- get status,
- get battery/power status,
- emit task lifecycle events.

Keep the protocol simple. A line-oriented text protocol is preferred unless practical implementation proves it inadequate.

## HTTP management

After the service layer and simulator exist, expose the same services through HTTP.

Initial management capabilities should include:

- status,
- task-list get/push,
- time setting,
- settings read/write,
- digest refresh/status,
- update trigger/staging.

The real-device web UI and simulator UI should reuse as much SPA code as practical, but this is not a requirement if it makes the device server unnecessarily heavy.

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
- launch simulator.

Avoid premature packaging complexity. A small standard-library-first Python toolset is preferred initially.

## Daily digest

Digest support follows the initial task/timer vertical slice.

Implementation steps:

1. download configured XTH URL into staged storage,
2. validate that the downloaded object is usable enough to replace the current copy,
3. atomically preserve/replace the last good digest,
4. parse the subset of XTH required by the existing daily digest,
5. render basic paginated reading screens,
6. add explicit refresh command,
7. hook opportunistic refresh into successful Wi-Fi client sessions.

Do not attempt full generic XTH compatibility before the actual digest renders correctly.

## Application updates

Keep update architecture minimal initially.

Possible baseline flow:

1. download candidate application payload to SD staging,
2. verify download completeness/integrity at least minimally,
3. promote candidate application code,
4. reboot/reload,
5. retain enough bootstrap behavior to recover from an application failure.

Exact rollback and bundle format remain open until the base runtime exists.

## Tests

Prioritize tests for code that can run under normal CPython:

- task parsing,
- task state transitions,
- timer pause/resume semantics,
- inactivity timeout behavior,
- optional wall-clock behavior,
- persistent state/storage failure handling,
- digest last-good-copy behavior,
- transport-independent service behavior.

Rendering can additionally use deterministic image snapshots where that is useful, but tests should not depend exclusively on pixel snapshots.

## Open implementation questions

These are intentionally unresolved until implementation or hardware experiments provide evidence:

- exact physical button mapping,
- which XTEink controls can wake from deep sleep,
- exact e-paper refresh strategy while a timer is running,
- internal grayscale representation and physical conversion/dithering strategy,
- exact configuration file format,
- exact serial wire protocol,
- exact HTTP endpoint names and response formats,
- application update bundle/rollback mechanism,
- how much XTH parsing is required for the current daily digest,
- whether the real-device management SPA should be embedded, stored on SD, or generated differently.

Do not promote these choices into `README.md` or `ARCHITECTURE.md` until they become durable decisions.

## Removal criteria

Delete this file when:

- the first-class components described in `ARCHITECTURE.md` have stable implementations,
- remaining work is represented by tests/issues/code comments rather than a broad implementation roadmap,
- this document no longer provides useful temporary sequencing information.
