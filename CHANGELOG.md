# Changelog

History of this repository and its three submodules, reconstructed from commit
logs in November 2026. None of the repositories has tagged releases, so entries
are grouped by date (UTC). The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## Unreleased

### Added
- Project documentation in the root repository: `ARCHITECTURE.md`,
  `BACKLOG.md`, `CHANGELOG.md`, `CLAUDE.md`, `DESIGN.md`, `OPERATIONS.md`,
  `PRD.md`, `SECURITY.md`, `TESTING.md`.

### Changed
- `README.md` rewritten with a system overview, repository layout and links to
  the documents above.

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
