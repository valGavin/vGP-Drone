# Backlog

Open work, from the code review of 2026-11-11. Nothing here has been started.

- **P0**: safety. Close before any further armed test.
- **P1**: needed for a flyable system.
- **P2**: cleanup and quality.

Repositories: **fw** = `speedybee_control`, **app** = `vgp_drone_controller`,
**cam** = `drone-camera`, **root** = this repository.

## P0

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-01 | fw | Packet timeout | If packets stop while WiFi stays associated, core 1 resends the last values forever and Betaflight never sees signal loss. Record the time of each valid packet; after a short silence stop sending iBUS. Depends on B-08. |
| B-02 | fw | Failsafe frame keeps the drone armed | `failsafe_channels` sets AUX1 to 1500, the armed value. Use 1200, or send nothing and let Betaflight's failsafe act. |
| B-03 | app | Leaving the app does not disarm | Quit calls `exit(0)` without sending a disarm packet, and nothing handles the app being paused or killed. Send disarm on Quit and on lifecycle pause. |
| B-04 | fw | Buffer overflow in `receive_phone_acknowledgment` | `strncpy(received_msg, p->payload, p->len)` into a 32-byte buffer, then a write at index `p->len`. Bound the copy to the buffer size. |
| B-05 | fw, app | Throttle floor below Betaflight's valid range | The app clamps throttle to 850 and starts at 855; Betaflight's default `rx_min_usec` is 885, and pulses below it count as invalid. Use 1000, or confirm `rx_min_usec` was lowered on the flight controller. |

## P1

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-06 | all | **Investigation: motors reach maximum and stop responding to throttle** | Reported by the owner. The Pico W was seen to receive and forward the values. Leading hypothesis: PID windup when armed without propellers, which is expected behaviour. Not confirmed. Next step: tests T-3 to T-5 in [TESTING.md](TESTING.md), and establish whether propellers were fitted. |
| B-07 | fw | Cross-core lock does not lock | `save_and_disable_interrupts` masks only the calling core. Protect `current_channels` with a `critical_section` or mutex. |
| B-08 | app | Fixed-rate sender | Packets are sent only from joystick callbacks, about 10 Hz and only while a stick moves. Send from a 50 Hz timer. |
| B-09 | app | Disarm does not reset throttle | `toggleArm` changes only AUX1; the stored throttle survives a disarm. Reset to minimum on disarm. |
| B-10 | cam | Camera announces itself once | `sendBroadcast()` runs once in `setup()`, then the sketch waits forever. If the app is not listening at that moment the camera never pairs. Repeat the broadcast inside `waitForPhoneAck()`. |
| B-11 | app | Tapping the ARM row does nothing | `settings_menu.dart`: `widget.onToggleArm;` is missing its call parentheses, so the row tap closes the menu without toggling. Only the switch works. |
| B-12 | fw, app | No sender check or pairing | Accept control packets only from the acknowledged phone address; add a pairing token and sequence number. See [SECURITY.md](SECURITY.md). |

## P2

| ID | Repo | Item | Detail |
|---|---|---|---|
| B-13 | fw | Non-standard iBUS header byte | Byte 1 is `0x2C`; the standard is `0x40`. Betaflight appears to accept it, but other parsers may not. |
| B-14 | fw | Per-frame `printf` | Printing 32 bytes of hex for every frame slows the loop below its nominal 143 Hz. Put it behind a debug flag. |
| B-15 | fw | Build is not standalone | `CMakeLists.txt` is a fragment that expects the `pico-examples` tree and `../wifi`. Make it a self-contained Pico SDK project. `dhcpserver/` is unused and can go. |
| B-16 | root | Stale submodule pointers | This repository pins `speedybee_control` at `c5b406c` (two commits behind, missing the multicore loop and failsafe) and `drone-camera` at `293c9ed` (one behind). `vgp_drone_controller` is pinned at `6f782e2`, also behind. Run `git submodule update --remote`, commit. |
| B-17 | fw, app | README corrections | fw README says UART0, the code uses UART1. app README has a placeholder clone URL (`yourusername`) and says the range is 1000 to 2000 where the code allows 850. |
| B-18 | cam | Stream handler edge cases | Frames 400 px wide or less leave the JPEG buffer pointer unset. The brownout detector is disabled, which hides power problems. |
| B-19 | app | No `android/` folder in the repository | The platform folder and manifest permissions are not committed, so the app cannot be built from a clean clone without regenerating them. |
| B-20 | fw | lwIP calls outside the lock | With `threadsafe_background`, calls such as `udp_sendto` from the main loop should be wrapped in `cyw43_arch_lwip_begin()` / `cyw43_arch_lwip_end()`. |
| B-21 | app | Own-IP detection assumes `192.168.` | `getOwnIP` returns `0.0.0.0` on hotspots using other private ranges. Neither firmware uses the value today. |
| B-22 | all | Automated tests | See "Tests to add" in [TESTING.md](TESTING.md). |
| B-23 | fw, cam | Credentials in source | Move SSID and password to an untracked header. See [SECURITY.md](SECURITY.md). |
| B-24 | root | Record the hardware | Frame, motors, props, battery, wiring, Betaflight version and a `diff all` dump are not recorded anywhere. |

## Suggested order

1. B-06, because its answer decides whether B-05 is urgent.
2. B-08, then B-01, B-02 and B-03 together: one change across app and firmware.
3. B-04 and B-07.
4. B-16, so this repository points at the fixed code.
