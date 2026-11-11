# Design

Why the system is built the way it is, what each choice costs, and where the
current design falls short. Rationale marked **(inferred)** was reconstructed
from the code and commit messages and should be corrected if wrong.

## Decisions

### D-1: WiFi over the phone's hotspot

The phone is both controller and access point, so nothing else is needed in
the field **(inferred)**.

Cost: range and reliability are those of a phone hotspot. Both firmwares carry
the hotspot credentials at compile time, so changing phone or SSID means
reflashing two devices.

### D-2: A Pico W presenting itself as an iBUS receiver

The flight controller stays stock Betaflight. The Pico W looks like an ordinary
serial receiver, so all stabilisation, mixing and ESC handling is left to
software that already works.

Cost: Betaflight only knows what the iBUS stream tells it. If the Pico W keeps
sending frames, Betaflight believes the link is healthy. Link-loss handling
therefore has to be correct on the Pico W: see D-9.

iBUS was chosen over SBUS, CRSF or others **(inferred)** because it is plain
non-inverted UART with a trivial checksum.

### D-3: UDP with broadcast discovery

UDP suits control data: a late packet is worthless, so retransmission is
unwanted. Broadcast discovery avoids typing IP addresses.

Cost: no delivery guarantee, no ordering, and as implemented no authentication
(see [SECURITY.md](SECURITY.md)).

### D-4: Fixed 10-byte binary control packet

Five `uint16` channels, little-endian, matching the in-memory layout on the
Pico W so the handler is a single `memcpy`.

Cost: no version field, sequence number or integrity check. Adding a channel
changes the length, and the receiver drops anything that is not exactly 10
bytes, so app and firmware must change together.

### D-5: iBUS loop on core 1

Commit `8936ba3` moved iBUS transmission to the second core at a fixed period
so the frame rate does not depend on packet arrival.

Cost: shared state crosses cores. It is guarded by a critical section.

### D-6: Incremental throttle

A touchscreen stick springs back to centre, which would cut throttle to
mid-range on release. The right stick therefore nudges a stored throttle value
up or down, and releasing it holds the value **(inferred)**.

Cost: throttle changes slowly (about 10 units per callback) and there is no
fast way to cut it other than disarming. Disarming returns it to idle.

### D-7: Arming on AUX1 from a menu switch

Arming is a deliberate two-tap action inside the settings menu rather than a
stick gesture, and the joysticks are ignored while disarmed.

Cost: disarming in a hurry also takes two taps. An always-visible disarm
control would be safer (B-27).

### D-8: Separate camera on its own microcontroller

Video is independent of control: MJPEG over HTTP from an ESP32-CAM straight to
the app. A camera fault cannot stall the control path.

Cost: a second WiFi client on the same hotspot and channel competes with
control packets for airtime. The app also refuses to open the control screen
until the camera is found.

### D-9: Heartbeat and a single failsafe policy

Added November 2026. The app sends the control packet every 20 ms from a
timer, whether or not a stick is moving. The Pico W treats 500 ms without a
valid packet, or loss of WiFi, as a lost link: it stops sending iBUS frames
and resets its channel values to disarmed defaults. Betaflight then sees
signal loss and applies its own failsafe.

The Pico W deliberately has no failsafe behaviour of its own beyond going
silent, so there is one policy, configured in one place (Betaflight).

Cost: safety now depends on Betaflight's failsafe being configured, and the
app and firmware must be updated together. When packets return the firmware
forwards them at once, including an armed AUX1 (B-26).

### D-10: The phone is identified by source address

Added November 2026. The Pico W records the address the acknowledgement came
from and accepts control packets only from it. It no longer trusts the address
written inside the `PHONE|<ip>` message.

Cost: this stops accidents, not attackers; an address can be forged (B-12).

## Known weaknesses

Each is tracked in [BACKLOG.md](BACKLOG.md).

| Area | Weakness | Item |
|---|---|---|
| Verification | The November 2026 changes have not been built or run on hardware | B-25 |
| Ranges | Throttle floor of 850/855 is below Betaflight's default `rx_min_usec` of 885 | B-05 |
| Trust | Source-address check only; no pairing token or sequence number | B-12 |
| Link recovery | Armed state is forwarded again as soon as packets return | B-26 |
| Arming UX | Disarm is two taps inside a menu | B-27 |

## Proposed direction

Not yet agreed; for the project owner to accept or reject.

1. **Packet v2.** Add a sequence number and a pairing token agreed during
   discovery (B-12).
2. **Standard ranges.** Throttle 1000 to 2000, with arming requiring throttle
   at 1000 (B-05).
3. **Re-arm policy.** Decide whether a link loss should require a fresh
   disarm before arming is forwarded again (B-26).
