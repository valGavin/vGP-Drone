# Product Requirements

> This document was written in November 2026 from the existing code and
> READMEs, after the fact. Items marked **(assumed)** are inferred and need
> confirming by the project owner.

## Purpose

Fly a self-built quadcopter from an Android phone, with live video, without a
conventional radio transmitter and receiver.

## Users

One: the builder. This is a personal hobby and learning project **(assumed)**,
not a product for others to fly.

## Goals

1. Control roll, pitch, yaw and throttle from on-screen joysticks.
2. Arm and disarm from the app.
3. See the drone's camera feed behind the controls.
4. Control the camera's flash LED brightness from the app.
5. Need no infrastructure beyond the phone: the phone's hotspot is the network.

## Non-goals

- Long range. WiFi from a phone hotspot limits range to tens of metres
  **(assumed acceptable)**.
- Autonomous flight, GPS modes, telemetry back to the phone.
- iOS support (launcher icons are configured for Android only).
- Low-latency FPV-grade video.

## Functional requirements

| ID | Requirement | State |
|---|---|---|
| F-1 | Pico W and camera discover the phone without manual IP entry | Done |
| F-2 | App sends roll, pitch, throttle, yaw and an arm channel to the Pico W | Done |
| F-3 | Pico W outputs a valid iBUS stream the flight controller accepts | Done (seen in Betaflight) |
| F-4 | App shows the MJPEG camera feed full-screen behind the controls | Done |
| F-5 | App shows current channel values | Done |
| F-6 | Flash LED brightness adjustable 0 to 255 | Done |
| F-7 | Camera feed can be reloaded after an error | Done |
| F-8 | Motors respond to throttle across the full range when armed | **Open** (BACKLOG B-06) |

## Safety requirements

These are what a flyable version needs. "Implemented" means the code exists
but has not been built or run on hardware (B-25).

| ID | Requirement | State |
|---|---|---|
| S-1 | If control packets stop for any reason, the drone disarms or enters Betaflight failsafe within 1 s **(threshold assumed)** | Implemented, unverified: firmware stops iBUS after 500 ms; the outcome then depends on Betaflight's failsafe settings |
| S-2 | Loss of WiFi results in a disarmed drone | Implemented, unverified: same mechanism as S-1 |
| S-3 | Closing or backgrounding the app disarms the drone | Implemented, unverified: Quit and leaving the foreground send disarm; S-1 covers a crash |
| S-4 | Malformed network input cannot crash or corrupt the receiver | Implemented; decoding is covered by host tests |
| S-5 | Only the paired phone can command the drone | Partly: source-address check only (B-12) |

## Hardware

| Part | Known | To confirm |
|---|---|---|
| Flight controller | SpeedyBee F7 V3 stack, Betaflight | Betaflight version, which UART the Pico W is wired to |
| ESC | BLHeli_32 4-in-1 (part of the stack) | Firmware version, motor protocol |
| Receiver | Raspberry Pi Pico W | How it is powered |
| Camera | ESP32-CAM, AI Thinker pinout | How it is powered |
| Frame, motors, props, battery | Not recorded | All |

## Success criteria

**(assumed)** A stable hover indoors, controlled from the app, with video, and
a drone that reliably stops when the link is lost.
