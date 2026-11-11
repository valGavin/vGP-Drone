# Backlog

Open work. Items closed by the November 2026 hardening round are listed in
[CHANGELOG.md](CHANGELOG.md); their IDs are not reused.

- **P0**: safety. Close before any further armed test.
- **P1**: needed for a flyable system.
- **P2**: cleanup and quality.

Repositories: **fw** = `speedybee_control`, **app** = `vgp_drone_controller`,
**cam** = `drone-camera`, **root** = this repository.

## P0

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-25 | all | Verify the hardening round on hardware | The November 2026 changes were written without a Pico SDK, Flutter or ESP32 toolchain: none of the three has been built or run. Build and flash all three, then run T-1 to T-9 in [TESTING.md](TESTING.md) with propellers off. The firmware and app must be updated together: the new firmware stops iBUS output when packets pause, and only the new app sends continuously. |
| B-05 | fw, app | Throttle floor below Betaflight's valid range | The app clamps throttle to 850 and idles at 855; Betaflight's default `rx_min_usec` is 885, and pulses below it count as invalid. Use 1000, or confirm `rx_min_usec` was lowered on the flight controller. Left unchanged because 850 was a deliberate choice; needs the owner's decision. |

## P1

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-06 | all | **Investigation: motors reach maximum and stop responding to throttle** | Reported by the owner. The Pico W was seen to receive and forward the values. Leading hypothesis: PID windup when armed without propellers, which is expected behaviour. Not confirmed. Next step: tests T-3 to T-5 in [TESTING.md](TESTING.md), and establish whether propellers were fitted. |
| B-12 | fw, app | Pairing token and sequence number | The firmware now accepts control packets only from the acknowledged phone's address, but a source address can be forged and packets can be replayed. Add a token agreed during discovery and a sequence number. This changes the wire format. |
| B-26 | fw | Decide the policy after a short dropout | After a link loss the firmware resumes forwarding as soon as packets return, including an armed AUX1. Alternative: require the app to send disarmed once before arming is forwarded again. Safer on the bench, but a brief WiFi hiccup in flight would then end in a disarm. Owner's decision. |
| B-27 | app | Always-visible disarm control | Disarming takes two taps inside the settings menu (DESIGN D-7). |

## P2

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-15 | fw | Build is not standalone | `CMakeLists.txt` is a fragment that expects the `pico-examples` tree and `../wifi`. Make it a self-contained Pico SDK project. `dhcpserver/` is unused and can go. |
| B-18 | cam | Brownout detector disabled | Disabling it hides power problems. Decide whether the camera's supply is good enough to turn it back on. |
| B-19 | app | No `android/` folder in the repository | The platform folder and manifest permissions are not committed, so the app cannot be built from a clean clone without regenerating them. |
| B-21 | app | Own-IP detection assumes `192.168.` | `getOwnIP` returns `0.0.0.0` on hotspots using other private ranges. Neither firmware uses the value. |
| B-22 | all | More automated tests | Host tests cover iBUS framing, packet decoding and the link timeout; a Dart test covers packet encoding. Still missing: stick mapping, arm/disarm state, camera. See [TESTING.md](TESTING.md). |
| B-23 | fw, cam | Credentials in source | Move SSID and password to an untracked header. See [SECURITY.md](SECURITY.md). |
| B-24 | root | Record the hardware | Frame, motors, props, battery, wiring, Betaflight version and a `diff all` dump are not recorded anywhere. |

## Suggested order

1. B-25: nothing else is trustworthy until the new code has run on the bench.
2. B-06, using the same bench session; its answer decides B-05.
3. B-26 and B-12, which need design decisions first.
