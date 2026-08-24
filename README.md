# XFlow

XFlow is a lightweight activity queue, timer, dashboard, and daily-digest viewer for the XTEink X4 e-paper device.

The project includes both the CircuitPython firmware that runs on the device and the host/development utilities used to push activities, synchronize time, receive device events, develop the UI, simulate the device without hardware, and exercise networking and transport behavior.

## Goals

- Run on the XTEink X4 using CircuitPython.
- Keep normal operation fully usable offline.
- Keep Wi-Fi and BLE disabled unless they are explicitly needed.
- Present a simple queue of time-boxed activities on the 800×480 e-paper display.
- Start every activity explicitly rather than automatically.
- Support Start, Pause/Resume, Stop, and activity completion.
- Keep wall-clock time optional; activity timing must work without RTC or network time.
- Allow activities and settings to be pushed over USB serial or HTTP.
- Support both Wi-Fi client mode and an explicit Wi-Fi AP mode for offline management.
- Provide first-class support for downloading and reading a daily digest in XTH format.
- Allow the replaceable application code to be updated without reflashing CircuitPython.
- Keep the application core independent from the physical display, buttons, radios, and transports.
- Run the same core and rendering code on desktop Python for development.
- Provide a browser-based simulator for the display, buttons, radios, serial transport, battery, and other device state.
- Include host-side tools for task push, time sync, serial/BLE events, notifications, digest workflows, and development workflows.

## Activity format

The activity queue is deliberately plain text and easy to edit by hand.

Each non-empty line represents one activity:

```text
<minutes>;<text>
```

Example:

```text
25;Review pull request
15;Email
45;Implement settings page
```

Blank lines are ignored. Lines beginning with `#` may be used as comments.

Activities do not contain real-world timestamps. Wall-clock time is runtime metadata obtained separately when available.

## Activity lifecycle

Activities are processed in file order.

A task never starts automatically. The user explicitly starts it.

### Idle

The next activity is displayed and its timer is stopped.

- `Start` begins the activity.

### Running

The activity timer is running.

- `Pause` pauses the timer.
- `Stop` cancels the current activity and advances to the next one.
- Timer completion completes the current activity and advances to the next one.

### Paused

The current activity remains selected but its timer is stopped.

- `Start` / `Resume` continues it.
- `Stop` cancels it and advances to the next activity.

## Daily digest

XFlow should provide first-class support for the existing daily digest as a second primary device function alongside the activity queue.

The digest is available as an XTH document from a direct HTTP(S) URL. Authentication may be embedded in the configured URL, so the device does not need a separate authentication flow for the baseline implementation.

Expected behavior:

- A configurable digest URL is stored in device settings.
- When Wi-Fi client connectivity is explicitly enabled and a connection succeeds, XFlow may automatically check/download the current digest.
- The user can also explicitly request a digest download/refresh.
- The latest successfully downloaded digest is stored locally and remains readable offline.
- A failed refresh must not discard the last usable local digest.
- Digest availability must not be required for the task/timer functionality.
- The application includes a basic XTH viewer suitable for the 800×480 e-paper screen.

The initial viewer only needs practical reading/navigation. Exact screen layout, pagination, navigation buttons, typography, and XTH feature coverage will be specified after the baseline application architecture is working.

Automatic digest download is tied to an already-requested Wi-Fi session; it is not a reason to keep Wi-Fi permanently enabled or periodically wake the radio by default.

## Time model

XFlow distinguishes between two kinds of time:

- **Monotonic time** is required for activity timers and must always work without network connectivity or a valid date/time.
- **Wall-clock time** is optional and is used only for displaying the real date/time and other features that explicitly need it.

The device has no requirement to know the real date/time immediately after boot or wake. The UI must remain fully usable while wall-clock time is unknown.

Wall-clock time may be synchronized from any available source, for example:

- NTP after connecting to a configured Wi-Fi network.
- A host utility over USB serial, typically using the host's current Unix timestamp.
- An HTTP API endpoint while the device is online.
- A browser management UI that sends the browser/host time to the device.

## Power management

Battery life is a first-class design constraint.

- Wi-Fi is off during normal activity operation unless explicitly requested.
- BLE is off during normal activity operation unless explicitly requested.
- An active task prevents inactivity shutdown.
- When no activity timer is running, user interaction resets an inactivity timer.
- The baseline inactivity timeout is approximately 15 minutes.
- After the final activity, the same inactivity policy applies and the device may enter deep sleep.
- Wake-up behavior and exact button mapping will be defined after testing the XTEink X4 hardware.

The device should expose battery voltage/level and USB/power status through the platform abstraction when available.

## Architecture

The application core is independent from physical I/O.

Conceptually:

```text
                    XFlow core
       tasks / digest / state / timer
                       |
             +---------+---------+
             |                   |
          rendering              I/O
             |                   |
      +------+-------+      +----+----------------+
      |              |      |                     |
 physical display   bitmap  physical buttons   transports
                            simulated buttons  serial / HTTP
```

The core must not need to know whether it is running on the physical XTEink device or on desktop Python.

## Rendering

Application screens render to a logical **800×480 grayscale surface**.

The same rendering code should be usable by multiple backends:

- XTEink display framebuffer/backend.
- Desktop bitmap/image backend.
- Browser simulator through rendered image frames.

The exact task, digest, settings, status, IP-address, and management screen layouts are intentionally not specified yet.

## Inputs and application services

Physical buttons and simulated buttons are alternative input sources for the same application commands.

Management transports should expose the same underlying application services rather than implementing separate behavior.

Expected operations include:

```text
push/get task list
get status
start
pause/resume
stop
set wall-clock time
get/update settings
refresh/get digest status
trigger application update
```

## USB serial

USB serial is a bidirectional management and event transport.

A host utility should be able to use it to:

- push activities,
- synchronize wall-clock time,
- query status and battery information,
- update settings,
- trigger application updates,
- receive asynchronous device events.

Example device events may include:

```text
TASK_STARTED
TASK_PAUSED
TASK_STOPPED
TASK_DONE
```

When connected to a Mac, a host helper can translate events such as `TASK_DONE` into native desktop notifications.

The exact serial wire format is intentionally left open for now; it should remain simple, human-readable where practical, and cheap to parse on CircuitPython.

## BLE events

BLE is an optional event transport, not something that needs to stay active continuously.

For low-power notification use cases, the preferred model is a short-lived BLE advertisement rather than maintaining a connection:

```text
task completes
    -> BLE on
    -> advertise event for a short interval
    -> BLE off
```

A host-side listener can continuously scan for XFlow advertisements and translate them into local notifications or other actions.

The advertisement payload only needs compact event metadata such as event type, device ID, task ID, and a sequence number. It does not need to carry the complete task list.

If USB serial is already connected, the implementation may prefer serial events and avoid enabling BLE entirely.

## Wi-Fi modes

Wi-Fi is optional during normal use and has three conceptual modes:

### Off

Default low-power operating mode.

### Client

The device connects to configured Wi-Fi credentials.

This can be used for:

- NTP time sync,
- HTTP management,
- downloading application updates,
- optional remote task-list retrieval,
- automatic daily-digest refresh when a digest URL is configured.

A successful client connection may perform opportunistic work such as NTP synchronization and digest refresh while the radio is already active, then Wi-Fi can be disabled again unless the user explicitly requested a persistent management session.

### Access point

The device explicitly starts its own Wi-Fi AP when requested.

This provides an offline-friendly management path when no usable Wi-Fi network exists. A phone or laptop can connect directly and use the same HTTP interface to push activities, configure the device, synchronize time, or inspect locally stored data.

AP mode does not imply Internet connectivity, so remote digest download is only available there if the AP/network arrangement actually provides an upstream path.

## HTTP server

The same lightweight HTTP API should be usable in both Wi-Fi client and AP modes.

The exact API is not final, but the initial shape may include endpoints such as:

```text
GET  /api/status
GET  /api/tasks
PUT  /api/tasks
POST /api/start
POST /api/pause
POST /api/stop
POST /api/time
GET  /api/settings
PUT  /api/settings
GET  /api/digest/status
POST /api/digest/refresh
POST /api/update
```

A small SPA can act as the management UI for both the physical device and the desktop simulator.

## Firmware bootstrap and updates

CircuitPython's `code.py` is a deliberately small bootstrap layer.

```text
code.py
   -> app.py
   -> XFlow application modules
```

`code.py` should change rarely. The normal OTA/update target is `app.py` and the application modules it owns.

Device-specific configuration is stored separately from application code. At minimum it should support:

- Wi-Fi SSID and password,
- application update source URL,
- daily-digest XTH source URL,
- optional task-list source URL,
- networking/power behavior settings.

A failed application update should not require reflashing CircuitPython merely to recover the device. More advanced rollback/integrity mechanisms can be added later.

## Desktop/browser simulator

The application should also run as a normal desktop Python program.

The simulator is intended to emulate the device environment, not merely render screenshots. Hardware/platform services should have controllable fake implementations for development.

At minimum this includes:

- display -> 800×480 bitmap/image buffer,
- buttons -> simulated button events,
- Wi-Fi -> fake off/client/AP modes, connection success/failure, IP address, and Internet availability,
- BLE -> fake enable/disable state and emitted advertisement events,
- USB serial -> fake bidirectional RX/TX stream,
- battery -> configurable charge level, voltage, USB/power state,
- wall clock -> known/unknown time and explicit time-sync events.

A lightweight development HTTP server hosts an SPA that shows the rendered device screen and provides controls for the simulated environment.

The SPA should be able to:

- press the physical-device buttons,
- inspect/change simulated Wi-Fi state,
- inspect BLE state and emitted advertisements,
- send data to the simulated device over fake serial,
- inspect serial output from the device,
- change battery/power state,
- make wall-clock time available/unavailable,
- emulate successful or failed remote downloads,
- expose useful debug/status information.

The browser remains a development surface around the real Python application. Task behavior, digest behavior, state machines, rendering, and service logic stay in the same Python code used by the physical device.

## Host and development utilities

This repository intentionally includes utilities for both **host machines** and **development machines**. They are part of XFlow rather than external throwaway scripts.

Expected utilities include:

- push an activity list over USB serial,
- automatically send the host's current Unix timestamp after connecting,
- query device status and battery state,
- configure Wi-Fi, digest source, and update settings,
- trigger or perform application updates,
- explicitly request a daily-digest refresh,
- listen for USB serial events,
- scan for short-lived XFlow BLE advertisements,
- show native macOS notifications for activity events,
- run the desktop/browser device simulator,
- render screens to image files for UI development and tests,
- exercise the application core and simulated transports without physical hardware.

The exact implementation language and packaging are not fixed yet. Simple Python utilities are preferred where they keep the device/host workflow easy to inspect and modify.

## Possible repository layout

```text
xflow/
├── README.md
├── code.py
├── app.py
├── xflow/
│   ├── core/
│   │   ├── tasks.py
│   │   ├── digest.py
│   │   ├── state.py
│   │   ├── timer.py
│   │   └── clock.py
│   ├── ui/
│   ├── services/
│   └── platform/
│       ├── circuitpython/
│       └── desktop/
├── host/
│   ├── push_tasks.py
│   ├── serial_listener.py
│   ├── ble_listener.py
│   └── notify.py
├── simulator/
│   ├── server.py
│   └── web/
├── tools/
└── docs/
```

The layout is intentionally provisional until the first implementation establishes natural module boundaries.

## Initial non-goals

The first version does not need:

- cloud accounts,
- an interactive digest authentication/login flow,
- synchronization between multiple XFlow devices,
- a general-purpose task-management system,
- rich editing directly on the keyboardless XTEink UI,
- continuous Wi-Fi or BLE connectivity,
- second-by-second e-paper refreshes,
- a sophisticated OTA bundle/rollback system,
- complete support for every possible XTH feature before the basic digest viewer works.

The first functional milestone is deliberately narrow:

**load activities -> display next activity -> Start -> Pause/Resume -> Stop/Complete -> advance.**

The next first-class milestone is:

**connect Wi-Fi on demand -> opportunistically download the configured XTH daily digest -> keep the last good copy -> read it in a basic on-device viewer.**
