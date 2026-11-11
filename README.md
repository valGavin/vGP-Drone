# vGP-Drone

A custom quadcopter flown from an Android phone over WiFi. The phone runs a
Flutter app that sends stick values to a Raspberry Pi Pico W on the drone; the
Pico W turns them into iBUS frames for a SpeedyBee F7 V3 flight controller
running Betaflight. An ESP32-CAM streams video back to the app.

> **Status: bench prototype, not flight-ready.** Control reaches the flight
> controller, but there are open safety items (see [BACKLOG.md](BACKLOG.md),
> P0) and an unresolved throttle issue. Do not arm with propellers fitted.

## How it fits together

```
Android app  --UDP/WiFi-->  Pico W  --iBUS/UART-->  SpeedyBee F7 V3  -->  ESC  -->  motors
     ^
     +------- MJPEG over HTTP ------- ESP32-CAM
```

The phone's WiFi hotspot is the network. The Pico W and the ESP32-CAM join it,
announce themselves by UDP broadcast, and the app replies to each.

## Repository layout

This repository is an umbrella. The code lives in three submodules:

| Path | What it is | Language / toolchain |
|---|---|---|
| [`speedybee_control`](https://github.com/valGavin/speedybee_control) | Pico W receiver firmware: UDP in, iBUS out | C, Pico SDK |
| [`vgp_drone_controller`](https://github.com/valGavin/vgp_drone_controller) | Android controller app | Dart, Flutter |
| [`drone-camera`](https://github.com/valGavin/drone-camera) | ESP32-CAM video stream and flash LED control | Arduino (ESP32 core 3.1.3) |

Clone with submodules:

```
git clone --recurse-submodules https://github.com/valGavin/vGP-Drone
```

The submodule pointers in this repository are behind the submodules' own
`master` branches (BACKLOG B-16). To work on the latest code:

```
git submodule update --remote
```

## Documentation

| Document | Contents |
|---|---|
| [PRD.md](PRD.md) | What the project is for, scope, requirements |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Components, protocols, ports, packet formats |
| [DESIGN.md](DESIGN.md) | Design decisions, their trade-offs, known weaknesses |
| [OPERATIONS.md](OPERATIONS.md) | Build, flash, configure, run, troubleshoot |
| [TESTING.md](TESTING.md) | Bench test procedure and test gaps |
| [SECURITY.md](SECURITY.md) | Threat model and known security issues |
| [BACKLOG.md](BACKLOG.md) | Prioritised open work |
| [CHANGELOG.md](CHANGELOG.md) | History across all four repositories |
| [CLAUDE.md](CLAUDE.md) | Working notes for AI-assisted development |

## Licence

LGPL-2.1. See [LICENSE](LICENSE).
