# Architecture

Describes the system as the code stands after the hardening round of
November 2026, at the submodule commits this repository pins.

## Components

```mermaid
flowchart LR
    subgraph Phone["Android phone (WiFi hotspot)"]
        App["vgp_drone_controller<br/>Flutter app"]
    end
    subgraph Drone
        Pico["Pico W<br/>speedybee_control"]
        FC["SpeedyBee F7 V3<br/>Betaflight"]
        ESC["BLHeli_32 ESC"]
        M["Motors"]
        Cam["ESP32-CAM<br/>drone-camera"]
    end
    App -- "UDP 9877: 10-byte control packet" --> Pico
    Pico -- "iBUS, UART 115200 8N1" --> FC
    FC --> ESC --> M
    Cam -- "HTTP 9878: MJPEG" --> App
    App -- "UDP 9879: brightness command" --> Cam
    Pico -. "UDP 8888 broadcast: drone announcement" .-> App
    Cam -. "UDP 8888 broadcast: camera announcement" .-> App
```

| Component | Repository | Main file | Role |
|---|---|---|---|
| Controller app | `vgp_drone_controller` | `lib/control.dart` | Joysticks, arming, video, sends control packets |
| Receiver | `speedybee_control` | `picow_speedybee_control.c` | Receives control packets, emits iBUS |
| Camera | `drone-camera` | `drone-camera.ino` | MJPEG stream, flash LED |
| Flight controller | not in this project | Betaflight configuration | Stabilisation and motor output |

## Network

The phone's hotspot is the access point. The SSID and password are compiled
into both firmwares (`WIFI_SSID` / `WIFI_PASSWORD` in the Pico W source, `ssid`
/ `password` in the camera sketch) and are blank in the repositories.

### Ports

| Port | Transport | Listener | Purpose |
|---|---|---|---|
| 8888 | UDP broadcast | App | Discovery: `DRONE\|<ip>` and `CAMERA\|<ip>` |
| 9877 | UDP | Pico W | Phone acknowledgement, then control packets |
| 9878 | UDP | ESP32-CAM | Phone acknowledgement |
| 9878 | TCP (HTTP) | ESP32-CAM | MJPEG stream at `/` |
| 9879 | UDP | ESP32-CAM | `BRIGHTNESS\|<0-255>` |

### Discovery and startup

1. Pico W and camera join the hotspot.
2. Each broadcasts its announcement on UDP 8888 until acknowledged: the
   Pico W every 2 s, the camera every second.
3. The app, listening on 8888, replies `PHONE|<phone ip>` to the sender's
   address: port 9877 for the drone, 9878 for the camera. It answers every
   announcement, since a device that is still announcing missed the last
   reply.
4. On acknowledgement the Pico W switches its port-9877 handler from
   acknowledgement to control packets and starts the iBUS loop on core 1. The
   camera starts its HTTP server.
5. When the app has seen both devices it waits 1 s and opens the control
   screen. It does not proceed with only one device.

The Pico W identifies the phone by the source address of the acknowledgement
datagram and ignores the address written inside it. The camera stores the
written address but does not use it. The app finds its own IP by picking the
first interface address starting with `192.168.`.

## Control packet (app to Pico W)

UDP, exactly 10 bytes, five little-endian `uint16` values. The Pico W drops a
packet unless it comes from the acknowledged phone's address, is exactly 10
bytes, and every value lies between 800 and 2200.

| Offset | Channel | Range | Notes |
|---|---|---|---|
| 0 | Roll | 1000 to 2000 | Left stick X, centre 1500 |
| 2 | Pitch | 1000 to 2000 | Left stick Y inverted, centre 1500 |
| 4 | Throttle | 850 to 2000 | Idles at 855; changed incrementally by right stick Y; returns to 855 on disarm |
| 6 | Yaw | 1000 to 2000 | Right stick X, centre 1500 |
| 8 | AUX1 | 1200 or 1500 | 1200 disarmed, 1500 armed |

The packet is copied over the first five entries of the Pico W's 14-channel
array. There is no sequence number or checksum. Encoding lives in
`lib/control_packet.dart`; decoding in `control_link.h`.

**Send rate.** The app sends the current values every 20 ms from a timer,
whether or not a stick is moving, plus once immediately on arm and disarm.
This stream is the link heartbeat. Stick values themselves still update only
as fast as the joystick widget calls back (the `flutter_joystick` default
period, about 100 ms).

**Disarm.** The app disarms, centres the sticks and idles throttle when the
ARM switch is turned off, when Quit is pressed, when the app leaves the
foreground (`hidden`, `paused`, `detached`), and when the control screen is
disposed.

**Throttle** is not a spring-centred axis. Each right-stick callback adds
`-y * 10` to the stored value, so full deflection changes throttle by about 10
units per callback. Releasing the stick holds the current throttle.

## iBUS output (Pico W to flight controller)

| Property | Value |
|---|---|
| UART | `uart1`, TX on GP4 (GP5 is defined as RX but never configured) |
| Format | 115200 baud, 8N1 |
| Rate | 7 ms sleep per loop: nominally 143 Hz. Frames are sent only while the control link is alive |

Frame, 32 bytes:

| Bytes | Content |
|---|---|
| 0 | `0x20` |
| 1 | `0x40` (servo command) |
| 2 to 29 | 14 channels, `uint16` little-endian |
| 30 to 31 | Checksum: `0xFFFF` minus the sum of bytes 0 to 29, little-endian |

Channels 1 to 5 come from the control packet. Channels 6 to 14 are fixed at
1500. Channel order is AETR, with arming on channel 5. Framing lives in
`ibus.h` and is checked by a host test against a reference frame. Setting
`IBUS_DEBUG_PRINT` to 1 prints every frame over stdio and slows the loop.

## Pico W execution model

| Core | Work |
|---|---|
| 0 | WiFi bring-up, discovery, then idle. lwIP callbacks (`pico_cyw43_arch_lwip_threadsafe_background`) write `current_channels` |
| 1 | Every 7 ms: decide whether the link is alive, copy `current_channels`, send an iBUS frame if it is |

The channel array, the time of the last valid packet and the "have a packet"
flag are shared between the cores and guarded by `channels_lock`, a critical
section.

**Link state and failsafe.** The control link is alive when WiFi reports
`CYW43_LINK_JOIN` and a valid control packet arrived within the last 500 ms
(`CONTROL_TIMEOUT_MS`). While it is not alive, core 1 sends nothing and resets
the channel array to the failsafe values (sticks centred, throttle 855, AUX1
1200, disarmed), so stale values are never sent. The flight controller sees
signal loss and applies its own failsafe. When valid packets return,
forwarding resumes at once (B-26).

## App structure

| File | Responsibility |
|---|---|
| `lib/main.dart` | App entry, dark theme, portrait lock |
| `lib/splash.dart` | Splash screen |
| `lib/connection.dart` | Hotspot reminder, discovery, acknowledgements |
| `lib/control.dart` | Channel state, fixed-rate UDP sender, arm/disarm, lifecycle handling, landscape control screen |
| `lib/control_packet.dart` | Channel constants and control packet encoding |
| `lib/widgets/dual_joystick.dart` | Stick-to-channel mapping; ignores input when disarmed |
| `lib/widgets/settings_menu.dart` | Reload camera, Quit, ARM switch |
| `lib/widgets/camera_feed.dart` | MJPEG view with reload on error |
| `lib/widgets/brightness_control.dart` | Flash LED slider |
| `lib/widgets/channel_value_overlay.dart`, `channel_value_bar.dart` | Channel read-out |
| `lib/widgets/device_status_box.dart` | Connection screen status tiles |

Dependencies: `flutter_joystick ^0.2.1`, `udp ^5.0.3`, `flutter_mjpeg ^2.0.4`.

## Camera

AI Thinker pin map. UXGA at JPEG quality 10 with two frame buffers when PSRAM
is present, otherwise SVGA at quality 12. The flash LED on GPIO4 is driven by
8-bit PWM at 5 kHz. The brownout detector is disabled at boot. The main loop
only polls for brightness packets; the HTTP server task serves the stream.
