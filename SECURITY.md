# Security

This is a hobby project that controls spinning propellers over WiFi. A security
flaw here is a physical safety problem.

## Trust model

Everything rests on the phone hotspot's WPA2 password. Any device that joins
the hotspot is treated as fully trusted: there is no authentication, pairing
or encryption above the WiFi layer.

## Known issues

| ID | Issue | Impact | Backlog |
|---|---|---|---|
| SEC-1 | The Pico W accepts control packets from any source address | Any host on the hotspot can command or arm the drone with a 10-byte UDP datagram to port 9877 | B-12 |
| SEC-2 | Stack buffer overflow in `receive_phone_acknowledgment`: `p->len` bytes are copied into a 32-byte buffer with no bound | A larger datagram sent to port 9877 before pairing corrupts memory on the receiver | B-04 |
| SEC-3 | Discovery is unauthenticated | A host can answer the broadcast first, or spoof `DRONE` / `CAMERA` announcements so the app sends to the wrong address | B-12 |
| SEC-4 | The camera stream is unauthenticated plain HTTP | Any host on the hotspot can watch the video | Not scheduled |
| SEC-5 | The brightness command is unauthenticated | Any host on the hotspot can drive the flash LED | Not scheduled |
| SEC-6 | No packet timeout | Not an attack by itself, but it turns any interruption of the app into a drone that keeps its last command | B-01 |
| SEC-7 | Control packets carry no sequence number | Captured packets can be replayed | B-12 |

## Credentials

The hotspot SSID and password are compiled into the Pico W firmware and the
camera sketch. They are blank in the repositories and must stay that way.

- Never commit a real SSID or password. Check `git diff` before every commit in
  `speedybee_control` and `drone-camera`.
- If credentials are committed by mistake, change the hotspot password. Deleting
  the commit does not remove it from clones or forks.
- A better arrangement is an untracked header (for example `secrets.h`, listed
  in `.gitignore`) with a committed `secrets.example.h`.

## Operating advice

- Use a strong, unique hotspot password, and do not use this hotspot for
  anything else while the drone is powered.
- Do not power the drone on a network you do not control.
- Check the hotspot's connected-device list: there should be exactly two
  clients.

## Reporting

Open a private security advisory on this repository (Security tab, "Report a
vulnerability"), or an issue if the problem is already public.
