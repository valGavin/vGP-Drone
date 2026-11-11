# Changelog

History of this repository and its three submodules, reconstructed from commit
logs in November 2026. None of the repositories has tagged releases, so entries
are grouped by date (UTC). The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Unreleased

Hardening round, 2026-11-11. **None of the code below has been built for its
target or run on hardware** (B-25); only the firmware's host tests have run.

### Added
- **speedybee_control:** control-link timeout: iBUS output stops after 500 ms
  without a valid control packet (B-01).
- **speedybee_control:** control packets accepted only from the acknowledged
  phone's address and only with plausible channel values (B-12, part).
- **speedybee_control:** host unit tests for iBUS framing, packet decoding and
  the link timeout; the logic moved to `ibus.h` and `control_link.h` (B-22,
  part).
- **vgp_drone_controller:** control packets sent every 20 ms from a timer
  (B-08).
- **vgp_drone_controller:** disarm on Quit, on leaving the foreground and on
  closing the control screen (B-03).
- **vgp_drone_controller:** `control_packet.dart` with a unit test, replacing
  the Flutter template test (B-22, part).

### Changed
- **speedybee_control:** on any link loss the firmware goes silent and resets
  its channels to disarmed defaults; it no longer sends a final frame with
  AUX1 at the armed value (B-02).
- **speedybee_control:** the phone is identified by the acknowledgement's
  source address, not the address inside the message.
- **speedybee_control:** iBUS command byte is the standard `0x40`, was `0x2C`
  (B-13).
- **speedybee_control:** per-frame hex dump is off unless `IBUS_DEBUG_PRINT`
  is set (B-14).
- **vgp_drone_controller:** disarming centres the sticks and returns throttle
  to idle (B-09).
- **vgp_drone_controller:** every discovery announcement is acknowledged, not
  only the first.
- **drone-camera:** the discovery broadcast repeats every second until
  acknowledged (B-10).
- **root:** submodule pointers moved to the hardening commits (B-16);
  documentation updated to match.

### Fixed
- **speedybee_control:** buffer overflow in the acknowledgement handler
  (B-04).
- **speedybee_control:** channel state shared between the cores is guarded by
  a critical section (B-07); lwIP calls from the main loop hold the lwIP lock
  (B-20).
- **vgp_drone_controller:** tapping the ARM row toggles arming (B-11).
- **vgp_drone_controller:** the control screen can no longer be opened twice
  when announcements keep arriving.
- **drone-camera:** frames 400 px wide or less no longer leave the JPEG
  buffer pointer unset (B-18, part).
- **speedybee_control, vgp_drone_controller:** README corrections (B-17).

## 2026-11-11

### Added
- **root:** project documentation: `ARCHITECTURE.md`, `BACKLOG.md`,
  `CHANGELOG.md`, `CLAUDE.md`, `DESIGN.md`, `OPERATIONS.md`, `PRD.md`,
  `SECURITY.md`, `TESTING.md`.

### Changed
- **root:** `README.md` rewritten with a system overview, repository layout
  and links to the documents above.

## 2025-07-04

### Fixed
- **root:** `.gitmodules` now lists the `drone-camera` submodule.

## 2025-04-05

### Added
- **speedybee_control:** WiFi-loss failsafe: on losing the link, channels are
  reset to defaults and iBUS output stops (`d7cd81e`).

## 2025-04-04

### Added
- **speedybee_control:** iBUS transmission on the second core at a nominal
  143 Hz; arming on channel 5 (`8936ba3`).
- **vgp_drone_controller:** ARM switch; joysticks disabled while disarmed
  (`1b57ff8`).

### Changed
- **speedybee_control:** iBUS moved to UART1 for stability; default throttle
  set to 855 (`8936ba3`).
- **vgp_drone_controller:** lowest throttle value set to 850 (`1b57ff8`).

## 2025-04-01

### Fixed
- **vgp_drone_controller:** the UDP sender is now initialised; before this the
  app sent nothing to the drone (`7cf8f9d`).

## 2025-03-23

### Added
- **vgp_drone_controller:** reload button shown when the camera feed errors
  (`2c4297d`).

### Changed
- **vgp_drone_controller:** widgets split into separate files (`c74e9bb`);
  new icon for brightness control (`2c4297d`).

## 2025-03-22

### Added
- **drone-camera:** initial upload: MJPEG stream, discovery broadcast, flash
  LED control (`293c9ed`).
- **vgp_drone_controller:** camera flash brightness control (`c032385`).
- **root:** `drone-camera` added as a submodule (`cec45ad`).

## 2025-03-19

### Added
- **root:** repository created; `speedybee_control` and `vgp_drone_controller`
  added as submodules (`cd4204d`).
- **speedybee_control:** initial upload of the Pico W receiver firmware
  (`2815d6e`).

Earlier `vgp_drone_controller` commits (before 2025-03-22) are not summarised
here.
