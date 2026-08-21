# Bosch Smart System — verified BLE field map

Reverse-engineered, ground-truth-verified decode of the **Bosch Smart System** eBike
Bluetooth LE data: the diagnostic telemetry channel (`0x0011`), the official Live Data
Interface (`eb21`), and how to capture and read both. Motor power, rider power, torque,
cadence, speed, energy, battery state, assist modes, and the on-board stored-log format —
each field labelled with a confidence marker and the evidence behind it.

**By [redundo.app](https://redundo.app).** © 2026 redundo.app. See [Credits](#credits).

> 📖 **New here? Start with [`docs/FINDINGS.md`](docs/FINDINGS.md)** — the plain-English
> walkthrough of what the system is and what we found. Then use [`docs/DECODER-CARD.md`](docs/DECODER-CARD.md)
> as the exact field-by-field reference.

> Verified on one bike, deeply: a **smart system (gen 4) Performance Line CX** — BDU3740,
> PowerTube 750, fw 20.x — cross-checked against Bosch Flow, an independent reference power
> meter, FIT ride logs, and Bosch's official LDI spec V1.0; plus **preliminary testing on a
> second bike (Performance Line SX, also gen 4)** that confirms the core carries across (see the
> CX-vs-SX section). Scope is deliberately honest — every row carries a confidence marker, and
> unproven guesses are labelled as such.

## What's here

| Path | Contents |
|------|----------|
| [`docs/FINDINGS.md`](docs/FINDINGS.md) | **Start here** — the plain-English walkthrough: what the system is, how it's framed, what the fields mean, the write channel, CX vs SX, and the capture method |
| [`docs/DECODER-CARD.md`](docs/DECODER-CARD.md) | **The decoder card** — the full field map with confidence markers, frame format, the boot-session capture insight, stored-log format, LDI + reference-meter channels, and a cross-project comparison |
| [`docs/BLE-ACCESS.md`](docs/BLE-ACCESS.md) | **BLE reachability of Remko's ~895-address registry** — which answer over BLE (174) vs refused (473, split by status code), the request/response grammar, and the auto-push / config-dump / poll-on-demand delivery model |
| [`docs/STARTUP-HANDSHAKE.md`](docs/STARTUP-HANDSHAKE.md) | **How the diagnostic session actually comes up** — the bike drives a staged MobileApp startup handshake (`0x40AA`/`0x40A9`, STAGE5→9). Answer it and you can subscribe to `0x0011` immediately; the notorious boot-window "kill" is a mute subscribe, not a timing problem. Confirmed on hardware; credit @Truuplei |
| [`docs/TWO-TRANSPORT-REACHABILITY.md`](docs/TWO-TRANSPORT-REACHABILITY.md) | **BLE vs USB are different roles, not a hierarchy** — the same bike read over both transports. 178 answer over BLE, 363 over USB, **400 by some transport**; 259 reachable over exactly one. USB gets the electrical internals, BLE gets the rider/HR/UI surface, and the charge-control surface is sealed on both |
| [`docs/NAV-PROTOCOL.md`](docs/NAV-PROTOCOL.md) | **The navigation / feature-streaming protocol** — the command-channel frames, the turn card + maneuver codes, the vector map-tile format, the z18 Web-Mercator coordinate system, the session handshake, and the cryptographic boundary on live display (companion to FINDINGS §6) |
| [`data/ble_accessibility.csv`](data/ble_accessibility.csv) | Every registry address × its BLE result (`value` / `supported-empty` / `not-available` / `command` / `passive`), keyed to Remko's `MCSP` address |
| [`data/example-readings-cx.csv`](data/example-readings-cx.csv) | **Example capture** — every field's actual reading from one CX request-sweep (158 fields), decoded/scaled where the scaling is known (unverified ones marked *guess*), serials/IDs masked with `x`. A concrete companion to the reachability map |
| [`data/diagnostic_fields.csv`](data/diagnostic_fields.csv) | `0x30` telemetry field IDs, scalings, confidence (channel `0011`) |
| [`data/ldi_fields.csv`](data/ldi_fields.csv) | Bosch LDI (`eb21`) fields — documented (spec V1.0) + undocumented |
| [`data/frame_types.csv`](data/frame_types.csv) | First-byte frame types |
| [`data/handshake_0x20.csv`](data/handshake_0x20.csv) | `0x20` handshake component-info codes |
| [`data/component_inventory.csv`](data/component_inventory.csv) | Example component inventory from a capture |
| [`data/fields.json`](data/fields.json) | All of the above as one machine-readable file |

## Summary

- **Three ways in — this map is the BLE diagnostic one.** The deep component data has two
  transports: **USB-C** to the drive unit — Remko Weijnen's
  [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader) documents all ~895 addresses
  that way (his repo also carries an unconfirmed, decompile-derived BLE transport; the
  hardware-verified BLE map is here) — and the **BLE diagnostic channel** `0x0011`, the focus here. Bosch's official
  **LDI** `eb21` is the third (telemetry only). Our contribution is the BLE half: of Remko's
  ~895 addresses, **which you can actually reach over Bluetooth, and how** (174 answer a request;
  473 refuse, and the refusals split into `DENIED` / `NO_ROUTE_FOUND` / `UNSUPPORTED` / `NOT_READY`
  rather than one wall; **19 more report only while riding → 193 report data**).
- **Two BLE surfaces, one vendor tree.** Both live under Bosch's base
  `0000xxxx-eaa2-11e9-81b4-2a2ae2dbcce4`: the **diagnostic channel** `0x0011` (the rich
  `0x30` telemetry) and the officially documented **LDI** `eb21`.
- **Frame format.** A notification packs length-prefixed frames; first byte is the type
  (`0x30` telemetry, `0x10` status, `0x20` handshake, `0x60/0x70` stored-log). A telemetry
  frame is a one-field protobuf: `30 <len> <idHi> <idLo> 08 <varint>`.
- **Fields are verified, not guessed.** Motor/rider power in watts (`98-5D`/`98-5B`),
  rider torque ÷20 = N·m (`98-14`, r = 1.000 vs LDI), cadence ÷2 (`98-5A`), speed ÷100 km/h
  (`98-2D`, integrates to odometer within 0.3%), delivered energy in Wh (`80-9C`), remaining
  energy ÷10 Wh (`80-91`), battery temp `zigzag(raw)/10` °C (`80-8B`), motor torque ÷20 N·m
  (`98-15`, peak = rated on two bikes). Any remaining candidates are marked as such.
- **The bike keeps its own ride summary.** Beyond the request-reachable 174, **19 fields report
  only while moving** (→ **193 report data**) — including the native `A2-4x` activity summary:
  avg/max rider power, avg/max cadence, avg speed, calories, moving time, trick stats, and
  **`A2-54 RIDER_ENERGY_SHARE`** — the bike's own rider-vs-motor energy split, validated to **±1–2%**
  of the integrated power. All auto-pushed for free in the passive stream.
- **Catch the bike's boot.** Which fields you get depends on *when you subscribe*, not the
  hardware — subscribe at power-on for the full set (both speed fields, cadence, and the
  stored-log transfer); join mid-session and you get a reduced set. Full note in the card.
- **⚠️ Don't integrate motor power raw.** `98-5D` is event-pushed and omits zeros — get
  energy from `80-9C`, or cadence-gate the motor stream first.

## Confidence markers

Every row is tagged: **✓** verified byte-exact on a BDU3740 · **✓ᵃ** verified byte-exact on
the Android capture · **?** plausible but unconfirmed · **✗** debunked. The CSV/JSON use the
words `verified` / `verified_android` / `candidate` / `debunked`.

## Cite this work

If you use this map, please cite **redundo.app** (a `CITATION.cff` is included, so GitHub's
"Cite this repository" button will do it for you). See [`LICENSE`](LICENSE) — the docs and
data are **CC BY 4.0** (free to use and build on, attribution required).

## Credits

Field map, method, and tooling by **[redundo.app](https://redundo.app)**, with thanks to:

- **Remko Weijnen — [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader)** — the
  parallel **USB-C** diagnostic project and its ~895-address BES3 registry (CC BY 4.0). Where
  this repo maps the BLE surface, his maps the USB one; we cross-reference field names to his
  registry and report which of his addresses are BLE-reachable.
- **Nik Leiser**, of the **[BikeBridge](https://codeberg.org/bg443/BikeBridge)** project (an
  open-source Android app for live BLE e-bike telemetry + commands) — contributed the HCI-snoop
  capture method and the Performance Line SX captures that opened the write/command channel and
  enabled the first cross-generation comparison.
- The earlier **community reverse-engineering posters** whose initial field notes gave this
  work its starting point.
- Cross-project confirmations from **Nilogax/SmartBridge**, **bestie-org/BEStie**,
  **RobbyPee**, **Xunil99/ha-bosch-ebike**, and the *"Nyon Unchained"* teardown (arXiv
  2404.12864). Details and specific agreements/disagreements are in the decoder card.

Full attribution in [`CREDITS.md`](CREDITS.md).

## Disclaimer

Interoperability and research documentation for owners and tinkerers. Not affiliated with or
endorsed by Robert Bosch GmbH. Modifying an eBike's configuration can affect its road-legal
status and warranty — know your local regulations before changing anything on a bike you ride.
