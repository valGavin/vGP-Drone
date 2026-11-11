# Operations

How to build, flash, configure and run the system. Steps marked **(unverified)**
are derived from the code and have not been re-run while writing this document.

> **Remove the propellers** for every step in this document.

## Prerequisites

| For | Needs |
|---|---|
| Pico W firmware | Pico SDK with CYW43/lwIP support, CMake, ARM GCC toolchain |
| Android app | Flutter with Dart SDK 3.7 or later, Android SDK |
| Camera | Arduino IDE with the ESP32 board package 3.1.3, a USB-serial adapter for the ESP32-CAM |
| Flight controller | Betaflight Configurator |

## 1. Get the code

```
git clone --recurse-submodules https://github.com/valGavin/vGP-Drone
cd vGP-Drone
```

Flash the Pico W and install the app from the same checkout. The firmware
stops driving the flight controller when control packets pause, and only an
app build with the fixed-rate sender sends them continuously.

## 2. Choose the hotspot credentials

Pick the SSID and password of the phone's hotspot. They are compiled into both
firmwares and are blank in the repositories.

- Pico W: `WIFI_SSID` and `WIFI_PASSWORD` in `speedybee_control/picow_speedybee_control.c`
- Camera: `ssid` and `password` in `drone-camera/drone-camera.ino`

**Do not commit real credentials.** See [SECURITY.md](SECURITY.md).

The Pico W connects with WPA2 mixed PSK. The app expects the phone's hotspot
address to start with `192.168.` (B-21).

## 3. Build and flash the Pico W

`speedybee_control/CMakeLists.txt` is a fragment, not a standalone project: it
has no `project()` or SDK initialisation, and it looks for `lwipopts.h` in a
sibling `../wifi` directory. It was written to sit inside the `pico-examples`
tree (B-15).

**(unverified)**

1. Copy or link `speedybee_control` into `pico-examples/pico_w/wifi/` and add
   `add_subdirectory(speedybee_control)` to that directory's `CMakeLists.txt`.
2. Configure with `-DPICO_BOARD=pico_w` and build the `picow_speedybee` target.
3. Hold BOOTSEL, plug in the Pico W, and copy `picow_speedybee.uf2` to the
   mounted drive.

The `dhcpserver/` directory is not referenced by the build. `ibus.h` and
`control_link.h` are header-only and need no build changes.

## 4. Flash the camera

1. Open `drone-camera/drone-camera.ino` in the Arduino IDE.
2. Board: AI Thinker ESP32-CAM. `CAMERA_MODEL_AI_THINKER` is selected in the
   sketch.
3. Hold GPIO0 low for upload, flash, then release and reset.

## 5. Build the app

The repository contains `lib/`, `assets/`, `test/` and `pubspec.yaml` but no
`android/` platform folder (B-19). **(unverified)**

```
cd vgp_drone_controller
flutter create --platforms android .
flutter pub get
```

Add to `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

The camera stream is plain HTTP. If the feed fails on a release build, allow
cleartext traffic for the app.

```
flutter run          # or: flutter build apk
```

## 6. Wire the Pico W to the flight controller

| Pico W | Flight controller |
|---|---|
| GP4 (UART1 TX) | RX pad of a free UART |
| GND | GND |
| Power | Not recorded: confirm the supply used |

Which UART on the SpeedyBee F7 V3 was used is not recorded.

## 7. Configure Betaflight

These settings follow from what the firmware sends. The values actually set on
the flight controller were not recorded, so check each one.

| Where | Setting |
|---|---|
| Ports | Serial RX on the UART wired to the Pico W |
| Receiver | Serial (via UART), provider IBUS |
| Receiver | Channel map AETR1234 |
| Modes | ARM on AUX1 with a range that includes 1500 and excludes 1200 |
| Failsafe | Stage 2 set to Drop. **Required:** when the link is lost the Pico W goes silent and this setting decides what the drone does |
| CLI | `get rx_min_usec`: the default is 885; the app sends throttle as low as 850 (B-05) |

## 8. Run

1. Turn on the phone hotspot.
2. Power the drone (props off).
3. Open the app and dismiss the hotspot reminder.
4. Wait for both the Drone and Camera tiles to show connected. The control
   screen opens one second later.
5. Open the settings menu (gear, top left) and tap ARM.
6. Left stick: roll and pitch. Right stick: yaw, and up/down to nudge throttle.
7. Disarm with the same control. Disarming returns throttle to idle. Quit and
   leaving the app also disarm, but disarm first anyway.

### Status LEDs

| Device | Pattern | Meaning |
|---|---|---|
| Pico W on-board LED | Toggles between connection attempts | Joining WiFi |
| Pico W on-board LED | Toggles every second | Announcing, waiting for the phone |
| Pico W on-board LED | Off | Phone acknowledged; sending iBUS |
| Camera flash LED | Fast blink | Joining WiFi |
| Camera flash LED | Slow blink | Waiting for the phone |
| Camera flash LED | Dim, steady | Streaming |

## Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Camera tile never connects | Wrong hotspot credentials, or the camera cannot join | Watch the camera's serial output |
| Drone tile never connects | Wrong hotspot credentials, or the Pico W cannot join | Watch the Pico W's USB serial output |
| App stuck on the connection screen with one device found | The app needs both devices | Power the missing device |
| No channel movement in Betaflight's Receiver tab | Wrong UART or provider, or TX/RX swapped | Recheck sections 6 and 7 |
| Will not arm, or RX loss flag | Throttle below `rx_min_usec` (B-05) | Check Betaflight's arming-disable flags |
| Motors climb to maximum and ignore the stick, props off | Expected PID behaviour without props; not yet confirmed as the cause here (B-06) | Follow [TESTING.md](TESTING.md), test T-4 |
| Receiver tab shows RX loss until the control screen opens | Expected: the Pico W sends iBUS only while control packets arrive | Open the control screen |
| RX loss whenever the sticks are idle | Old app build without the fixed-rate sender | Rebuild the app from the current code |
| Drone keeps running after the app closes | Old firmware, or Betaflight failsafe not configured | Disconnect the battery; reflash and check section 7 |
| Camera feed error | Stream dropped | Reload Camera in the settings menu |

## Emergency stop

Do not rely on a software stop: none has been verified on hardware (B-25). On
the bench, be ready to disconnect the battery. Disarming in the app, quitting
it, or turning the hotspot off should each stop the motors, the last two
through Betaflight's failsafe, which depends on the flight controller's
failsafe configuration.
