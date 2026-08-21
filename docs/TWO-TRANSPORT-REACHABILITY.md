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

Scope, as everywhere in this repo: **one bike, deeply verified.** A second bike, or
a different head-unit / battery configuration, would shift the absent-hardware rows.
