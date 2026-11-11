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
therefore has to be correct on the Pico W (it is not yet: B-01, B-02).

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

Cost: shared state crosses cores and needs a real lock (B-07).

### D-6: Incremental throttle

A touchscreen stick springs back to centre, which would cut throttle to
mid-range on release. The right stick therefore nudges a stored throttle value
up or down, and releasing it holds the value **(inferred)**.

Cost: throttle changes slowly (about 10 units per callback), there is no fast
way to cut it other than disarming, and disarming does not reset it (B-09).

### D-7: Arming on AUX1 from a menu switch

Arming is a deliberate two-tap action inside the settings menu rather than a
stick gesture, and the joysticks are ignored while disarmed.

Cost: disarming in a hurry also takes two taps. An always-visible disarm
control would be safer.

### D-8: Separate camera on its own microcontroller

Video is independent of control: MJPEG over HTTP from an ESP32-CAM straight to
the app. A camera fault cannot stall the control path.

Cost: a second WiFi client on the same hotspot and channel competes with
control packets for airtime. The app also refuses to open the control screen
until the camera is found.

## Known weaknesses

Each is tracked in [BACKLOG.md](BACKLOG.md).

| Area | Weakness | Item |
|---|---|---|
| Link loss | No packet timeout; stale values are resent indefinitely | B-01 |
| Link loss | Failsafe frame holds AUX1 at the armed value | B-02 |
| App lifecycle | Quit, crash or backgrounding leaves the drone armed at its last throttle | B-03 |
| Input handling | Acknowledgement handler can overflow a 32-byte stack buffer | B-04 |
| Ranges | Throttle floor of 850/855 is below Betaflight's default `rx_min_usec` of 885 | B-05 |
| Concurrency | Interrupt masking used as a cross-core lock | B-07 |
| Control rate | App sends about 10 Hz and only while a stick moves | B-08 |
| Discovery | Camera announces itself only once | B-10 |
| Trust | Any host on the hotspot can command the drone | B-12 |

## Proposed direction

Not yet agreed; for the project owner to accept or reject.

1. **Heartbeat.** The app sends the control packet at a fixed rate (50 Hz)
   from a timer, whether or not a stick is moving. This is the foundation for
   everything else here.
2. **Receiver timeout.** The Pico W records the arrival time of each valid
   packet. After a short silence (250 to 500 ms) it stops sending iBUS frames,
   which lets Betaflight's own failsafe act, rather than inventing a second
   failsafe policy on the Pico W.
3. **Disarm on exit.** The app sends a disarm packet on Quit and on lifecycle
   pause. The timeout in step 2 covers the cases where it cannot.
4. **Packet v2.** Add a sequence number and a pairing token agreed during
   discovery; the Pico W accepts packets only from the acknowledged phone
   address.
5. **Standard ranges.** Throttle 1000 to 2000, with arming requiring throttle
   at 1000.
