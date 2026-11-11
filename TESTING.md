# Testing

## Current state

There are no automated tests that exercise this project's logic.

| Repository | Tests |
|---|---|
| `speedybee_control` | None |
| `vgp_drone_controller` | One file, `test/widget_test.dart`, the size of the Flutter template; not reviewed or run |
| `drone-camera` | None |

All verification so far has been manual: USB serial output from the Pico W
(every iBUS frame is printed in hex) and Betaflight Configurator.

## Bench procedure

**Propellers off for every test.** Run the tests in order; each one clears a
layer for the next.

| ID | Test | Steps | Pass when |
|---|---|---|---|
| T-1 | Discovery | Hotspot on, power the drone, open the app | Both tiles connect and the control screen opens |
| T-2 | Control packets reach the Pico W | Watch the Pico W serial output while moving each stick | Bytes for channels 1 to 4 change with the matching stick; channel 5 changes with ARM |
| T-3 | iBUS reaches Betaflight | Receiver tab, move each stick | Roll, Pitch, Throttle, Yaw and AUX1 bars follow the app, including throttle coming back **down** |
| T-4 | ESCs and motors | Motors tab, raise and lower the master slider | All four motors follow the slider both ways |
| T-5 | Armed behaviour | Arm from the app, hold throttle steady, watch the Motors tab | Record whether the four outputs stay together or diverge |
| T-6 | WiFi loss | Armed at low throttle, turn the hotspot off | Motors stop |
| T-7 | App loss | Armed at low throttle, force-stop the app | Motors stop within 1 s |
| T-8 | Camera | Open the control screen | Video shows; Reload Camera recovers a dropped stream |
| T-9 | Flash LED | Move the brightness slider | LED brightness follows |

T-7 is expected to **fail** until B-01 is fixed, and T-6 leaves the drone armed
in the failsafe frame (B-02). Keep a hand on the battery connector.

### Reading T-3 to T-5 for the throttle issue (B-06)

| Result | Conclusion |
|---|---|
| T-3 fails: the throttle bar sticks or shows RX loss | The fault is in the app or Pico W; start with the throttle floor (B-05) |
| T-3 passes, T-4 fails | ESC or motor problem |
| T-3 and T-4 pass, outputs diverge in T-5 | PID windup with no propellers: expected Betaflight behaviour, not a fault |

Status on 2026-11-11: the owner reports that testing in Betaflight Configurator
was fine. Which of T-3 and T-4 that covered, and whether propellers were fitted
during the original runaway, is not yet recorded.

## Tests to add

In rough priority order. None exists yet.

| Target | What to test | How |
|---|---|---|
| iBUS framing | Header, channel packing, checksum against a known-good frame | Move `send_ibus_packet`'s packing into a pure function and unit test it on the host |
| Control packet parsing | Wrong lengths rejected; values clamped | Host unit test of the handler's parsing |
| Acknowledgement parsing | Oversized and unterminated input handled (B-04) | Host unit test |
| Stick mapping | `details.x/y` to channel values, including clamps and inversion | Dart unit test of a function extracted from `dual_joystick.dart` |
| Packet encoding | Byte layout of the 10-byte packet | Dart unit test of a function extracted from `control.dart` |
| Arm state | Disarm resets throttle; disarm sent on exit | Widget test once B-03 and B-09 are implemented |
| Link-loss timeout | iBUS stops after the configured silence | Bench test with a logic analyser on GP4, after B-01 |

## Before any flight

All of these must hold:

- Every P0 item in [BACKLOG.md](BACKLOG.md) is closed.
- T-1 to T-9 pass.
- Betaflight failsafe is configured and T-6 and T-7 have been seen to stop the
  motors.
