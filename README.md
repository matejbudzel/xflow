# XFlow

XFlow is an offline-first activity timer, dashboard, and document reader for the XTEink X4 e-paper device.

The project includes the CircuitPython device application together with host and development utilities used to manage, simulate, inspect, and integrate the device.

## Fundamental principles

- **Offline first.** Core activity and reading features must remain useful without Wi-Fi, BLE, wall-clock time, or a host computer.
- **Battery first.** Wi-Fi and BLE stay disabled unless they are explicitly needed. A running activity prevents automatic sleep; otherwise inactivity may put the device into deep sleep.
- **Portable core.** Application state, behavior, and screen composition are independent from physical hardware. The same core should run on the XTEink X4 and on desktop Python.
- **Abstract I/O.** Physical buttons, simulated buttons, USB serial, HTTP, BLE events, display output, time, networking, battery state, and persistent storage are accessed through platform abstractions.
- **One logical display.** Application screens target an 800×480 grayscale surface. The physical device backend presents it on e-paper; development backends render it to bitmap images.
- **Multiple management transports.** USB serial and HTTP expose the same underlying application services. HTTP may be available while the device is a Wi-Fi client or while it provides its own access point.
- **Optional wall-clock time.** Activity timing uses monotonic time and does not depend on real date/time. Wall-clock time may be learned opportunistically from NTP, USB serial, HTTP, or a browser.
- **SD-backed mutable state.** Internal flash is primarily for CircuitPython, bootstrap code, and executable application code. User data, configuration, downloaded content, persistent state, caches, and staged updates belong on the SD card when available.
- **Recoverable bootstrap.** CircuitPython's `code.py` stays small and stable and launches replaceable application code. Missing or damaged mutable storage should lead to a recoverable diagnostic state rather than a boot loop.
- **Host tooling is part of the product.** The repository contains portable Python utilities for task push, time synchronization, status inspection, updates, serial/BLE event handling, notifications, simulation, and development workflows.

## Primary features

### Activity flow

Activities are a simple ordered queue of explicitly started time boxes. The baseline human-editable format is:

```text
<minutes>;<text>
```

Example:

```text
25;Review pull request
15;Email
45;Implement settings page
```

The user can Start, Pause/Resume, Stop, or complete the current activity. Stopping cancels the current activity and advances to the next one.

### Daily digest

XFlow has first-class support for downloading and reading a daily digest in XTH format from a configurable HTTP(S) URL. Authentication may be embedded in the URL.

When Wi-Fi client connectivity is already requested and succeeds, XFlow may opportunistically synchronize time and refresh the digest. The latest good digest remains available offline, and failed refreshes must not discard it.

### Host integration

USB serial is bidirectional and can be used for management, time synchronization, status queries, and asynchronous events. Short-lived BLE advertisements may also carry events such as activity completion so a host helper can produce native notifications without keeping BLE connected.

### Development simulator

The desktop/browser simulator runs the real application core while replacing hardware with controllable fakes. It should simulate the 800×480 display, buttons, Wi-Fi modes, BLE, USB serial, battery/power state, wall-clock availability, persistent storage, and remote-download success or failure.

## Documentation

- [`ARCHITECTURE.md`](ARCHITECTURE.md) describes the intended system design and component relationships, whether or not every part is implemented yet.
- [`IMPLEMENTATION.md`](IMPLEMENTATION.md) tracks the temporary implementation plan and unresolved near-term work. It is expected to shrink and eventually disappear.
- [`AGENTS.md`](AGENTS.md) defines repository-wide rules for coding agents and contributors.

The README intentionally avoids implementation detail. Durable structural decisions belong in `ARCHITECTURE.md`; temporary sequencing and incomplete work belong in `IMPLEMENTATION.md`.
