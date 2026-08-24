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

The SD card is the preferred home for persistent mutable state:

```text
/sd/
    config/
    tasks/
    digest/
    state/
    cache/
    updates/
```

Expected categories include:

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
- the minimum bootstrap/recovery material required to start without the SD card.

### Simulator

Desktop Python uses the same logical storage API backed by a local directory such as `sim-data/`.

The simulator must also be able to emulate absent, read-only, or failing storage.

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
- expose parsed content to a basic on-device XTH viewer.

Digest fetching must not require Wi-Fi to remain enabled continuously.

## Rendering

Application screens target a logical **800×480 grayscale surface**.

UI code should describe what is drawn without depending on whether the output destination is physical e-paper or a desktop bitmap.

Backends include:

- physical XTEink display backend,
- desktop bitmap/image backend,
- browser simulator frame delivery.

The display backend owns any conversion required by the actual e-paper panel, including monochrome conversion or dithering if needed.

The application must not rely on frequent refreshes. E-paper updates should be deliberate and tied to meaningful state changes or appropriately coarse timer updates.

## Input model

Application commands are independent from their physical source.

Possible input sources include:

- physical XTEink buttons,
- simulator/browser buttons,
- USB serial commands,
- HTTP API calls.

A button press and an equivalent remote command should reach the same application service rather than duplicate behavior in separate code paths.

Exact physical button mapping is intentionally outside this architecture until the interaction design is finalized.

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
trigger or stage application update
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

The wire protocol should remain simple and cheap to parse on CircuitPython. Human readability is preferred where it does not complicate the implementation.

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
- application update download,
- optional remote task retrieval,
- HTTP management.

When the requested work is complete, Wi-Fi may be disabled again unless the user requested a persistent management session.

### Access point

The device explicitly creates its own Wi-Fi network for local offline management.

A phone or laptop can connect directly and use the same HTTP management interface for tasks, settings, time synchronization, local data inspection, and locally supplied updates.

AP mode does not imply Internet access.

## HTTP management

The same lightweight HTTP service should work in client and AP modes.

A management SPA can use it to configure and inspect the real device.

The exact endpoint names are implementation details, but the API should map directly onto the transport-independent application services.

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

Exact deep-sleep wake sources and button behavior are hardware-dependent and should be finalized only after testing the XTEink X4 implementation.

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

- display -> 800×480 bitmap buffer,
- buttons -> synthetic input events,
- Wi-Fi -> off/client/AP state, IP address, connectivity success/failure,
- Internet -> controllable remote-request success/failure,
- BLE -> enable/disable state and captured advertisements,
- USB serial -> bidirectional fake stream,
- battery -> configurable voltage/level/USB state,
- wall clock -> known/unknown state and explicit sync,
- storage -> local directory plus injected failure modes.

A lightweight local HTTP server hosts a browser SPA around the simulated device. The browser is a development control surface, not a second implementation of XFlow behavior.

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
