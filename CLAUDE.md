# CLAUDE.md

Notes for Claude when working in this project.

## What this is

A phone-controlled quadcopter. Android app (Flutter) to Raspberry Pi Pico W
(C, Pico SDK) over UDP, Pico W to a SpeedyBee F7 V3 running Betaflight over
iBUS, plus an ESP32-CAM video stream. Read [ARCHITECTURE.md](ARCHITECTURE.md)
before changing anything.

Status: bench prototype with open safety items. See [BACKLOG.md](BACKLOG.md).

## Working agreement

- The owner acts as project manager: architecture, goals and concept are their
  decisions. Claude acts as the engineer: writes the code and proposes best
  practices and alternatives.
- Propose before making a design change. Implement once it is agreed.
- Say plainly when something is unverified. Most of this project can only be
  verified on hardware Claude cannot reach.

## Layout

This repository holds documentation and submodule pointers only. Code changes
go to the submodule's own repository, then the pointer here is bumped.

| Path | Repository | Stack |
|---|---|---|
| `speedybee_control/` | `valGavin/speedybee_control` | C, Pico SDK, lwIP |
| `vgp_drone_controller/` | `valGavin/vgp_drone_controller` | Dart, Flutter |
| `drone-camera/` | `valGavin/drone-camera` | Arduino, ESP32 core 3.1.3 |

Default branch is `master` in all four. The pointers here are stale (B-16):
run `git submodule update --remote` to see current code.

## Rules

1. **Safety first.** This code drives motors. Never weaken or remove a
   failsafe, timeout or arming check without the owner's explicit agreement.
   Any instruction that involves arming says "propellers off".
2. **Never commit credentials.** `WIFI_SSID`, `WIFI_PASSWORD`, `ssid` and
   `password` stay blank in the repositories.
3. **The wire format is shared.** The 10-byte control packet, the ports and
   the discovery strings are defined in more than one repository. Change them
   in every repository in the same piece of work and update
   [ARCHITECTURE.md](ARCHITECTURE.md).
4. **Do not claim hardware behaviour that was not observed.** Code that
   compiles is not code that flies. State what was run and what was not.
5. **Keep the documents current.** A change that closes a backlog item removes
   it from [BACKLOG.md](BACKLOG.md) and adds a line to
   [CHANGELOG.md](CHANGELOG.md).

## Conventions

- Commit subjects use the tags already in the history: `[FEAT]`, `[FIX]`,
  `[CHG]`. Use `[DOC]` for documentation.
- C: Pico SDK style as in `picow_speedybee_control.c`; doc comments above
  functions.
- Dart: `flutter_lints`; one widget per file under `lib/widgets/`.
- One backlog item per pull request where practical; reference its ID.

## Commands

Build steps are in [OPERATIONS.md](OPERATIONS.md). Several are marked
unverified: the Pico W build expects the `pico-examples` tree (B-15) and the
app has no committed `android/` folder (B-19).

```
git submodule update --init --remote     # fetch current submodule code
cd vgp_drone_controller && flutter analyze && flutter test
```

## Facts worth knowing before editing

- Ports: UDP 8888 discovery, UDP 9877 control, 9878 camera (UDP ack and HTTP
  stream), UDP 9879 brightness.
- Channel order AETR; AUX1 1200 is disarmed, 1500 is armed.
- Throttle is incremental, not spring-centred.
- On the Pico W, core 0 runs lwIP callbacks and core 1 sends iBUS.
  `save_and_disable_interrupts` does not protect state shared between them.
- There is no packet timeout yet (B-01). Treat every armed test as if the
  drone may not stop.
