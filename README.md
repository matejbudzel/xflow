# XFlow

XFlow is a lightweight activity queue, timer, and dashboard for the XTEink X4 e-paper device.

The project includes both the CircuitPython firmware that runs on the device and the host/development utilities used to push activities, synchronize time, receive device events, develop the UI, and simulate the device without hardware.

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
- Allow the replaceable application code to be updated without reflashing CircuitPython.
- Keep the application core independent from the physical display, buttons, radios, and transports.
- Run the same core and rendering code on desktop Python for development.
- Provide a browser-based simulator for the display and buttons.
- Include host-side tools for task push, time sync, serial/BLE events, notifications, and development workflows.

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
              tasks / state / timer
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

The exact task, settings, status, IP-address, and management screen layouts are intentionally not specified yet.

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
- optional remote task-list retrieval.

### Access point

The device explicitly starts its own Wi-Fi AP when requested.

This provides an offline-friendly management path when no usable Wi-Fi network exists. A phone or laptop can connect directly and use the same HTTP interface to push activities, configure the device, or synchronize time.

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
- optional task-list source URL,
- networking/power behavior settings.

A failed application update should not require reflashing CircuitPython merely to recover the device. More advanced rollback/integrity mechanisms can be added later.

## Desktop simulator

The application should also run as a normal desktop Python program.

Hardware services are replaced by desktop implementations or stubs:

- display -> bitmap/image buffer,
- buttons -> simulated button events,
- Wi-Fi -> stub,
- BLE -> stub or development implementation,
- battery -> configurable simulated state.

A lightweight development HTTP server hosts an SPA that shows:

- the rendered 800×480 screen,
- controls corresponding to physical buttons,
- optional debug/status information.

The browser is only a view and input surface. The real XFlow core, state machine, and renderer remain in Python.

## Host and development utilities

This repository intentionally includes utilities for both **host machines** and **development machines**. They are part of XFlow rather than external throwaway scripts.

Expected utilities include:

- push an activity list over USB serial,
- automatically send the host's current Unix timestamp after connecting,
- query device status and battery state,
- configure Wi-Fi and update settings,
- trigger or perform application updates,
- listen for USB serial events,
- scan for short-lived XFlow BLE advertisements,
- show native macOS notifications for activity events,
- run the desktop device simulator,
- render screens to image files for UI development and tests,
- exercise the application core without physical hardware.

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
- authentication,
- synchronization between multiple XFlow devices,
- a general-purpose task-management system,
- rich editing directly on the keyboardless XTEink UI,
- continuous Wi-Fi or BLE connectivity,
- second-by-second e-paper refreshes,
- a sophisticated OTA bundle/rollback system.

The first functional milestone is deliberately narrow:

**load activities -> display next activity -> Start -> Pause/Resume -> Stop/Complete -> advance.**
