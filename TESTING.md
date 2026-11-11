# Testing

## Current state

| Repository | Tests | Last result |
|---|---|---|
| `speedybee_control` | `test/test_host.c`: iBUS framing against a reference frame, control packet decoding and rejection, link timeout including counter wrap-around | Passed on 2026-11-11 (host, gcc) |
| `vgp_drone_controller` | `test/control_packet_test.dart`: control and disarm packet encoding | **Never run**: no Flutter toolchain was available when it was written |
| `drone-camera` | None | |

```
cd speedybee_control && gcc -Wall -Wextra -I. test/test_host.c -o test_host && ./test_host
cd vgp_drone_controller && flutter analyze && flutter test
```

Nothing from the November 2026 hardening round has been built for its target
or run on hardware (BACKLOG B-25). Earlier verification was manual: USB serial
output from the Pico W and Betaflight Configurator. To see iBUS frames on the
Pico W's serial output, set `IBUS_DEBUG_PRINT` to 1.

## Bench procedure

**Propellers off for every test.** Run the tests in order; each one clears a
layer for the next.

| ID | Test | Steps | Pass when |
|---|---|---|---|
| T-1 | Discovery | Hotspot on, power the drone, open the app | Both tiles connect and the control screen opens |
| T-2 | Control packets reach the Pico W | Build with `IBUS_DEBUG_PRINT` set to 1; watch the serial output while moving each stick | "Control link up" appears when the control screen opens; bytes for channels 1 to 4 change with the matching stick; channel 5 changes with ARM |
| T-3 | iBUS reaches Betaflight | Receiver tab, move each stick | Roll, Pitch, Throttle, Yaw and AUX1 bars follow the app, including throttle coming back **down** |
| T-4 | ESCs and motors | Motors tab, raise and lower the master slider | All four motors follow the slider both ways |
| T-5 | Armed behaviour | Arm from the app, hold throttle steady, watch the Motors tab | Record whether the four outputs stay together or diverge |
| T-6 | WiFi loss | Armed at low throttle, turn the hotspot off | Motors stop |
| T-7 | App loss | Armed at low throttle, force-stop the app | Motors stop within 1 s |
| T-7a | Quit and background | Armed at low throttle: press Quit; repeat with the Home button | Motors stop at once both times |
| T-7b | Link recovery | After T-6, turn the hotspot back on without touching the app | Record what happens: the firmware forwards packets again as soon as they return (B-26) |
| T-7c | Idle link | Control screen open, disarmed, sticks untouched for a minute | Receiver tab shows a steady signal, no RX loss |
| T-8 | Camera | Open the control screen | Video shows; Reload Camera recovers a dropped stream |
| T-9 | Flash LED | Move the brightness slider | LED brightness follows |

T-6 and T-7 depend on Betaflight's failsafe settings as well as the firmware,
and neither has been run since the firmware changed. Keep a hand on the
battery connector.

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

In rough priority order (B-22).

| Target | What to test | How |
|---|---|---|
| Stick mapping | `details.x/y` to channel values, including clamps and inversion | Dart unit test of a function extracted from `dual_joystick.dart` |
| Arm state | Disarm resets throttle; disarm sent on Quit and on leaving the foreground | Widget test of the control screen with a fake sender |
| Acknowledgement parsing | Oversized and unterminated input handled | Host unit test, once the parsing is moved into a pure function |
| Link-loss timing | iBUS stops 500 ms after the last packet | Bench test with a logic analyser on GP4 |

## Before any flight

All of these must hold:

- Every P0 item in [BACKLOG.md](BACKLOG.md) is closed.
- T-1 to T-9 pass.
- Betaflight failsafe is configured and T-6 and T-7 have been seen to stop the
  motors.
