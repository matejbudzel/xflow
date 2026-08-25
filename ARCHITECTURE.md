# XFlow Architecture

This document describes the intended architecture of XFlow. It is a target design, not an implementation-status report. Components described here may exist only partially or not at all yet.

## System boundaries

XFlow consists of three cooperating environments:

1. **Device runtime** — CircuitPython running on the XTEink X4.
2. **Host utilities** — portable Python tools running on macOS, Debian, or another normal Python environment.
3. **Development simulator** — the same application core running on desktop Python with simulated hardware and a browser control surface.

The device runtime is the primary product. Host tools and the simulator are first-class parts of the same repository because they define how the device is configured, fed data, debugged, tested, and integrated with the desktop.

## Layering

The application is divided conceptually into four layers:

```text
+---------------------------------------------------------+
|                    Application UI                       |
| task screens / digest viewer / settings / status       |
+---------------------------------------------------------+
|                    Application Core                     |
| tasks / state machines / timers / digest / commands    |
+---------------------------------------------------------+
|                     Services                            |
| storage / clock / networking / updates / event output  |
+---------------------------------------------------------+
|                 Platform Adapters                       |
| XTEink hardware or desktop simulator implementations   |
+---------------------------------------------------------+
```

Application behavior must not depend directly on CircuitPython hardware APIs. Platform-specific code provides implementations of narrow interfaces consumed by the core and services.

## Boot and application lifecycle

CircuitPython starts from a deliberately small `code.py` bootstrap.

```text
CircuitPython
    -> code.py
        -> bootstrap/recovery
            -> app.py
                -> XFlow modules
```

`code.py` should change rarely. Its responsibilities should remain limited to actions necessary to start or recover the replaceable application.

Normal updates replace application code rather than requiring CircuitPython to be reflashed.

The bootstrap must tolerate failure conditions such as:

- missing SD card,
- unreadable or corrupt persistent storage,
- missing application files,
- failed or incomplete staged update.

Such failures should result in a recoverable diagnostic or management state rather than a permanent crash loop.

## Persistent storage

Mutable data is stored through a storage abstraction. Application code must not scatter direct `/sd/...` accesses throughout the codebase.

### Physical device

The SD card is the preferred home for persistent mutable state. The baseline layout is:

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

The exact set of state/cache files may evolve, but configuration has a stable conceptual home at `/sd/system/configuration.json`.

Expected mutable categories include:

- Wi-Fi credentials and device settings,
- current task queue,
- task progress or other durable runtime state when needed,
- downloaded XTH digest,
- caches,
- staged application updates.

Internal flash is primarily for:

- CircuitPython,
- `code.py`,
- executable application code,
- static management assets that are part of the application build,
- the minimum bootstrap/recovery material required to start without the SD card.

### Simulator

Desktop Python uses the same logical storage API backed by a local directory such as `sim-data/`.

The simulator must also be able to emulate absent, read-only, or failing storage.

## Configuration

Device configuration is JSON stored through the storage abstraction, with `/sd/system/configuration.json` as the physical-device baseline.

Configuration should cover at least:

- Wi-Fi SSID and password,
- Wi-Fi/session behavior,
- inactivity timeout,
- daily digest URL,
- application update manifest URL,
- BLE event behavior,
- display full-refresh interval.

Secrets and authentication-bearing URLs are runtime data and must not be committed to Git.

## Time model

XFlow distinguishes strictly between **monotonic time** and **wall-clock time**.

### Monotonic time

Required for activity timing, inactivity shutdown, retries, and internal durations. It must work without network connectivity and without knowledge of the real date or time.

### Wall-clock time

Optional metadata used for user-facing date/time and features that explicitly need civil time.

Wall-clock time may become known after boot through one of several sources:

- NTP during a successful Wi-Fi client session,
- a Unix timestamp sent over USB serial by a host utility,
- an HTTP time-setting endpoint,
- browser time supplied through the management UI.

The application must remain functional when wall-clock time is unknown.

## Activity model

The baseline task-list format is deliberately human-editable:

```text
<minutes>;<text>
```

Tasks are processed in file order and are not tied to real-world timestamps.

The core state machine contains at least:

- **Idle** — next/current task selected, timer not running.
- **Running** — activity timer advancing.
- **Paused** — current task retained, timer stopped.

Core transitions include:

```text
Idle --Start--> Running
Running --Pause--> Paused
Paused --Resume--> Running
Running --Stop--> next task / Idle
Paused --Stop--> next task / Idle
Running --Complete--> next task / Idle
```

Stopping cancels the current task and advances to the next task.

The task state machine must be independent from display refresh frequency, wall-clock time, networking, and host connectivity.

## Daily digest

The daily digest is a second primary application function alongside the activity flow.

A configurable HTTP(S) URL points directly to an XTH document. Authentication may be embedded in that URL.

The digest subsystem should:

- download a new digest when explicitly requested,
- optionally refresh opportunistically during an already-requested Wi-Fi client session,
- preserve the last successfully downloaded digest if refresh fails,
- keep the digest readable offline,
- expose content to a basic on-device XTH viewer.

Digest fetching must not require Wi-Fi to remain enabled continuously.

### XTH handling

CrossPoint Reader is the primary behavioral/file-format reference for XTH support on the XTEink X4. XFlow does not depend on its C/C++ implementation, but its parser and viewer are useful evidence for the file structure and constrained-device access patterns.

XTH content may be large and must not be assumed to fit into RAM. Parsing and page access should prefer streaming/chunked reads and SD-backed indexes/caches where useful.

XTH can contain 2-bit page data. XFlow converts/dithers source grayscale into its shared 1-bit logical framebuffer before display, so the physical X4 and browser simulator render the same final pixels.

Initial XTH support should implement the subset required by the current daily digest rather than attempting a generic document-engine rewrite.

## Rendering

Application screens target a logical **800×480 1-bit black/white framebuffer**.

There is one rendering result regardless of destination:

```text
UI/content
    -> optional grayscale/source decoding
    -> shared dithering/threshold pipeline
    -> 800x480 1-bit framebuffer
    -> physical display OR browser bitmap
```

Dithering is therefore not a device-backend detail. The simulator must show the same 1-bit result that the physical e-paper backend receives.

### E-paper refresh policy

Screen composition is separate from physical refresh policy.

Every committed screen-render request increments a global partial-refresh counter. The default policy is:

- use partial/fast refreshes for normal updates,
- force a full refresh after 15 partial refreshes,
- reset the counter after a full refresh,
- allow the full-refresh interval to be configured globally.

This policy is system-wide rather than tied to a particular screen such as the task timer or digest reader. Any screen may request that the next presentation be forced to a full refresh when necessary.

The physical display adapter owns the actual waveform/driver calls, while the application-facing presentation service owns the global refresh-cycle policy.

The default of 15 is intentionally aligned with proven XTEink/CrossPoint behavior and can be changed after hardware testing if XFlow's UI update patterns need a different trade-off.

The application must not rely on second-by-second display refreshes. Timer rendering should remain coarse enough for e-paper and battery constraints.

## Input model

Application commands are independent from their physical source.

### Physical controls

The XTEink X4 application-visible controls are modeled as seven buttons:

```text
Bottom area:  2 x 2 buttons
Right side:   2 buttons
Right side:   Power button
```

The hardware reset control is not treated as an application input.

Exact action mapping may evolve with interaction design, but these physical positions are stable input identities and should be mirrored by the simulator.

### Simulator controls

The browser simulator should present controls in approximately the same spatial arrangement as the physical X4 rather than as an unrelated generic button list. This makes UI/input behavior easier to reason about before hardware testing.

A physical button press, simulated button press, serial command, and equivalent HTTP operation should reach the same application service rather than duplicate behavior in separate code paths.

## Management services

Transport-independent services expose operations such as:

```text
get status
get/push task list
start
pause/resume
stop
set wall-clock time
get/update settings
get battery/power state
refresh/get digest status
check/stage/apply application update
```

USB serial and HTTP adapt these services to different transports.

## USB serial

USB serial is bidirectional.

It supports host-to-device management such as:

- pushing tasks,
- sending current Unix time,
- querying status,
- changing settings,
- triggering updates.

It also supports device-to-host events such as:

```text
TASK_STARTED
TASK_PAUSED
TASK_STOPPED
TASK_DONE
```

The exact wire protocol may evolve during implementation. It should follow conventional line-oriented request/response framing where practical, remain cheap to parse on CircuitPython, and keep asynchronous event messages unambiguous.

## BLE event output

BLE is normally disabled.

For low-power host notifications, XFlow may briefly enable BLE and broadcast a compact advertisement event instead of creating or maintaining a BLE connection.

Example flow:

```text
task completes
    -> BLE on
    -> advertise TASK_DONE for a short interval
    -> BLE off
```

A desktop helper may continuously scan for these advertisements and convert them into native notifications.

If USB serial is already connected, the device may prefer serial event delivery and avoid enabling BLE.

## Wi-Fi

Wi-Fi has three conceptual modes:

### Off

Default low-power mode.

### Client

The device connects to configured infrastructure Wi-Fi. A successful session may be used for:

- NTP time synchronization,
- digest refresh,
- application update check/download,
- optional remote task retrieval,
- HTTP management.

When the requested work is complete, Wi-Fi may be disabled again unless the user requested a persistent management session.

### Access point

The device explicitly creates its own Wi-Fi network for local offline management.

A phone or laptop can connect directly and use the same HTTP management interface for tasks, settings, time synchronization, local data inspection, and locally supplied updates.

AP mode does not imply Internet access.

## HTTP management

The same lightweight HTTP service works in client and AP modes and listens on port 80 unless platform constraints require otherwise.

The management API is REST-style and uses JSON for structured request/response payloads.

Endpoint naming is not architectural, but the service surface should map directly onto transport-independent application services rather than duplicating behavior.

A static `index.html`/SPA is part of the application build and is served by the real device. It calls the same REST endpoints used by other HTTP clients. The device does not generate the SPA dynamically.

The desktop simulator may serve the same management SPA plus additional simulator-only controls for fake hardware state.

## Application build and updates

Application updates are manifest-driven and originate from a configurable HTTP(S) URL.

The development/build machine produces a deployable application artifact set and publishes it through an ordinary static HTTP server such as nginx. XFlow does not depend on nginx specifically; it only requires HTTP access to the configured manifest and files.

The update manifest should contain at least:

- a release/build identifier,
- a list of files belonging to the application build,
- download paths/URLs,
- integrity metadata such as file size and/or hash.

A build identifier may initially be timestamp-based as long as ordering/comparison is deterministic.

Baseline update flow:

```text
user requests update check
    -> download manifest
    -> compare release/build ID
    -> offer/update if newer
    -> download files to SD staging
    -> verify staged files
    -> promote application build
    -> restart application/device
```

The management SPA should expose at least an update-check action and, when an update exists, an explicit apply/update action.

`code.py` remains outside ordinary application replacement. The bootstrap must keep enough recovery behavior to survive an incomplete or broken application promotion. The exact amount of retained previous-version data may evolve, but staging and verification happen before promotion.

Static management SPA assets are part of the same application build manifest as the Python application code.

## Power management

Battery life is a first-class design constraint.

Rules:

- Wi-Fi stays off unless requested.
- BLE stays off unless requested or briefly needed to emit an event.
- A running task prevents inactivity shutdown.
- When no task timer is running, user activity resets an inactivity deadline.
- The baseline inactivity timeout is approximately 15 minutes.
- After the last task, the same inactivity policy applies.
- Deep sleep is preferred when the device becomes inactive long enough.

Battery voltage/level and USB/power state are exposed through the platform abstraction when available.

### Wake source

The baseline design uses the **Power button as the only required deep-sleep wake control**. XFlow should not spend implementation complexity on waking from the six normal UI buttons unless later evidence provides a strong reason to support it.

This matches the established interaction model of existing XTEink X4 firmware such as CrossPoint Reader: normal controls are application inputs while Power owns the sleep/wake lifecycle.

## Battery abstraction and simulation

The platform battery service exposes at least:

- estimated level/percentage when available,
- measured voltage when available,
- USB/external-power state when available.

The browser simulator provides convenient battery presets rather than requiring raw values for every test. At minimum:

- full,
- medium,
- low,
- critical.

USB/charging/external-power state should be independently switchable. The simulator may additionally allow exact voltage/percentage overrides for edge-case testing.

## Host utilities

Host and development utilities are portable Python programs. They should run on macOS and Debian without platform-specific rewrites whenever practical.

Expected responsibilities include:

- task push over USB serial,
- time synchronization,
- status and battery inspection,
- settings management,
- update triggering/staging,
- serial event listening,
- BLE advertisement scanning,
- native host notifications where supported,
- simulator startup,
- screen rendering/testing utilities.

Platform-specific notification integration may be isolated behind a small adapter while the rest of the helper remains portable Python.

## Desktop/browser simulator

The simulator runs the real core, services, and UI code under normal desktop Python.

Platform dependencies are replaced with controllable fake adapters:

- display -> exact 800×480 1-bit application framebuffer rendered as an image,
- buttons -> seven synthetic controls arranged like the physical X4,
- Wi-Fi -> off/client/AP state, IP address, connectivity success/failure,
- Internet -> controllable remote-request success/failure,
- BLE -> enable/disable state and captured advertisements,
- USB serial -> bidirectional fake stream,
- battery -> full/medium/low/critical presets plus USB/power state and optional exact overrides,
- wall clock -> known/unknown state and explicit sync,
- storage -> local directory plus injected failure modes.

A lightweight local HTTP server hosts a browser SPA around the simulated device. The browser is a development control surface, not a second implementation of XFlow behavior.

## Runtime compatibility testing

Portable application logic is primarily unit-tested on normal CPython because that gives the fastest development loop and best tooling.

A second smoke-test layer should run portable/device-intended Python under CircuitPython's native Unix port when practical. This is not hardware emulation and does not validate XTEink I/O, but it can catch language/runtime assumptions that work on CPython and fail under the CircuitPython/MicroPython runtime family.

The Unix port is not a complete substitute for the real CircuitPython board build and omits some CircuitPython-specific modules. Therefore the intended confidence ladder is:

```text
CPython unit/integration tests
    -> CircuitPython Unix-port compatibility smoke tests
        -> real XTEink X4 hardware tests
```

Hardware adapters remain covered by focused real-device tests where desktop/native runtimes cannot represent the behavior.

## Event flow

Application events should be produced once in the core and then routed to interested outputs.

Conceptually:

```text
core event
    -> UI state/update
    -> serial event sink, if available
    -> BLE advertisement sink, if enabled/appropriate
    -> simulator/debug sink
```

This keeps event semantics independent from transport availability.

## Dependency direction

Dependencies should point inward:

```text
platform adapters -> services -> core
UI/render backend -> application state/core
transports -> application services
```

Core modules must not import CircuitPython-specific hardware modules, simulator web-server modules, or host-tool code.

## Reference implementations

CrossPoint Reader is a useful external reference for XTEink-specific behavior, including:

- X4 physical controls,
- power/sleep interaction,
- XTH/XTCH parsing and streaming access,
- SD-backed caching on the ESP32-C3,
- periodic partial/full e-paper refresh policy.

XFlow should reuse proven constraints and file-format knowledge where useful, but it remains an independent Python/CircuitPython architecture rather than a port of CrossPoint's C/C++ application structure.

## Repository shape

The exact layout may evolve, but the intended separation is approximately:

```text
xflow/
    core/
    ui/
    services/
    platform/
        circuitpython/
        desktop/
host/
simulator/
tools/
docs or root architecture documents
```

Module boundaries should remain small and explicit rather than allowing large all-purpose files to accumulate.
