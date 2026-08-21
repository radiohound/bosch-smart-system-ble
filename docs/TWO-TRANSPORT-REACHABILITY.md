# Two-transport reachability: BLE vs USB are different roles, not a hierarchy

`data/ble_accessibility.csv` now carries a second reachability column,
`usb_2026_08_21`, from a direct USB-C read of the same bike (BDU3740 / Performance
Line CX, PowerTube 750, one Kiox 300, one battery). The USB path is the drive
unit's diagnostic bus, reached here through the port on the LED controller
(BRC3600) using Remko Weijnen's [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader).

Reading both columns together produces the headline result: **the two transports
are not a privilege hierarchy — they authenticate as different bus roles, and
each is granted a slice the other is refused.**

## The numbers (one bike, 895 registry addresses)

| | count |
|---|---|
| readable over **BLE** (178 = 174 plain-read + 4 UDAM RPCs needing a ConfigId) | 178 |
| readable over **USB** | 363 |
| readable over **both** | 141 |
| **BLE-only** | 37 |
| **USB-only** | 222 |
| **union — reachable by SOME transport** | **400**  (45% of 895) |

**259 of the 400 reachable fields — nearly two-thirds — are reachable over exactly
one transport.** Which transport you use is not a footnote; it decides access to
most of the surface. Neither transport is a superset of the other.

## The two roles

- **USB = a diagnostic / service role.** It is granted the hardware internals a
  BLE client is refused: `PRESENT_CELL_VOLTAGE`, `PRESENT_DISCHARGE_CURRENT`,
  `MAXIMUM_CHARGING_CURRENT`, the MAX/MIN pack-temperature extremes, and the long
  tail of factory/service data. It is **refused** the rider/session surface.
- **BLE bonded phone = the MobileApp role.** It is granted the rider/activity/UI
  data a diagnostic tool is refused, and it is the role that *provides* data to the
  bike. It is **refused** the electrical internals.

The 37 BLE-only fields cluster unmistakably by theme — every one is MobileApp
domain:

- rider-session: every `*_FOR_RIDER` (state of charge, energy), `ACTIVITY_ID`,
  `TIME_ZONE_OF_ACTIVITY`, `ASSIST_MODE_USAGE_*`, `BRAKE_EVENTS`,
  `AUTOMATIC_ACTIVITY_RESET`
- heart rate: `AVERAGE_HEART_RATE`, `MAXIMUM_HEART_RATE`
- display / UI: `VIEW_STRIPE_*`, `SUPPORTED_TILE_IDS` / `SUPPORTED_TILE_SIZES`,
  `UNLOCK_TOKENS_NONCE`, `AVAILABLE_BUTTONS`, `ACTIVE_UI_PRIORITY`, `KEY_DEVICE`
- rider-facing state: `BIKE_NAME`, `REACHABLE_RANGE`, `BIKE_NOT_MOVING`,
  `ASSIST_MODE_LIMITS`

This directly explains why HR, altitude, navigation, and assist-mode configuration
are all reachable over BLE but not over the diagnostic USB port: they belong to the
MobileApp role.

## A third band, sealed on every transport

One group is refused on **both** transports — `DENIED` over BLE and, tested with a
charger connected and actively charging, `DENIED` over USB too: the charge-**control**
surface — `CHARGING_SETTINGS`, `CHARGING_INFORMATION`, `SO_C_UPPER_LIMIT` /
`SO_C_LOWER_LIMIT`, `REMAINING_CHARGING_TIME`. A native charge limit exists in the
schema, but it is reachable from no client interface we can find. `DENIED` is policy
applied to the address before routing, so this is a hard wall, not a readiness state.

## `DENIED` is verb-independent, not just read-refused

A `DENIED` answer to a READ only proves the *read verb* was refused. MCSP has others,
and on this bus some datapoints refuse a one-shot read while answering a subscribe —
`DriveUnit.REACHABLE_RANGE` is the known case over BLE, and Flow itself only ever
reads it via `.subscribe()`. So the wall above needed testing with a second verb
before it could be called a wall at all.

It has been. **Every one of the 417 addresses that answers `DENIED` to a READ over USB
was re-tried with SUBSCRIBE. All 417 refused that too** — a well-formed
`SUBSCRIBE_RESPONSE` carrying `DENIED`, not a dropped frame. Zero exceptions, none
left untested. Positive control in the same session: `PRESENT_CELL_VOLTAGE`,
`PRESENT_PACK_TEMPERATURE` and `STATE_OF_CHARGE` all subscribed and pushed live values
(pack temp read 23.0 °C and the subscription pushed 23.1 °C — a real notification, not
an echo), so the subscribe path demonstrably works on this bike.

Per-address results are the `usb_subscribe_2026_08_21` column.

**A `SUBSCRIBE_RESPONSE: ok` is NOT evidence of reachability.** 44 subscribes were
accepted and then never pushed anything — and every single one came from an address
whose read said `NO_ROUTE_FOUND` or `NOT_READY`, never from a `DENIED` one. 39 of the
44 are on hardware this bike does not have (no second battery, no ABS, no connect
module). **The bike accepts subscribe registrations for absent components and simply
never delivers.** Anyone scoring subscribe-acks as reachable would report 44 fields
that do not exist.

So the two verbs answer the same question the same way, and the charge-control surface
is sealed against both.

## Method notes, so this is reproducible

- **Join key:** match the CSV's `mcsp` column (`0x00D8`, component-qualified) against
  the USB tool's raw address — NOT the `ble_id` (`80-D8`), whose component byte uses
  a different convention.
- **Re-probe a "no reply" before recording it as a denial.** A ~900-address sweep
  drops the occasional frame; three fields first flagged BLE-only turned out to read
  over USB on a targeted re-probe (`REGIO_SPEED_APPLICATION_AVAILABLE`,
  `CURRENT_MANIFEST`, `BOOTLOADER_ERROR_STATES`). The 37 BLE-only figure is after
  that correction.
- `usb_2026_08_21` values: `value`, `DENIED`, `NO_ROUTE_FOUND`, `NOT_READY`,
  `UNSUPPORTED`, `timeout`. `NO_ROUTE_FOUND` is the only decline that signals a
  genuinely absent component (no ABS, no second battery); the rest prove the
  component is present.
- `usb_subscribe_2026_08_21` values: `DENIED` (or another status name) = the bike
  refused the SUBSCRIBE; `accepted-silent` = subscribe accepted, nothing pushed inside
  the notify window; `value` = accepted and a value arrived; empty = not attempted,
  because the plain READ already succeeded. Only addresses that *declined* a read were
  retried.
- **The frame-drop rate is real and symmetric — never update a cell from a single
  read.** Across two full sweeps of the same parked bike, 11 addresses moved `DENIED`
  → `timeout` and 11 moved `timeout` → `DENIED`, ~1.5% each way. `Battery.SERIAL_NUMBER`
  timed out once, which is proof enough that a lone timeout means nothing. Every
  timeout in the subscribe pass was re-probed individually until it produced a real
  status; that is why the column contains no `timeout` values.

Scope, as everywhere in this repo: **one bike, deeply verified.** A second bike, or
a different head-unit / battery configuration, would shift the absent-hardware rows.
