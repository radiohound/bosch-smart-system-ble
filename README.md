# Bosch Smart System — verified BLE field map

Reverse-engineered, ground-truth-verified decode of the **Bosch Smart System** eBike
Bluetooth LE data: the diagnostic telemetry channel (`0x0011`), the official Live Data
Interface (`eb21`), and how to capture and read both. Motor power, rider power, torque,
cadence, speed, energy, battery state, assist modes, and the on-board stored-log format —
each field labelled with a confidence marker and the evidence behind it.

**By [redundo.app](https://redundo.app).** © 2026 redundo.app. See [Credits](#credits).

> Verified on one bike, deeply: smart system **BDU3740 / Performance Line CX** (fw 20.x,
> PowerTube 750), cross-checked against Bosch Flow, an independent reference power meter,
> FIT ride logs, and Bosch's official LDI spec V1.0. Scope is deliberately honest — every
> row carries a confidence marker, and unproven guesses are labelled as such.

## What's here

| Path | Contents |
|------|----------|
| [`docs/DECODER-CARD.md`](docs/DECODER-CARD.md) | **The decoder card** — the full field map with confidence markers, frame format, the boot-session capture insight, stored-log format, LDI + reference-meter channels, and a cross-project comparison |
| [`docs/CAPTURE-GUIDE.md`](docs/CAPTURE-GUIDE.md) | How to capture a ride with the Bosch Decoder Android app |
| [`data/diagnostic_fields.csv`](data/diagnostic_fields.csv) | `0x30` telemetry field IDs, scalings, confidence (channel `0011`) |
| [`data/ldi_fields.csv`](data/ldi_fields.csv) | Bosch LDI (`eb21`) fields — documented (spec V1.0) + undocumented |
| [`data/frame_types.csv`](data/frame_types.csv) | First-byte frame types |
| [`data/handshake_0x20.csv`](data/handshake_0x20.csv) | `0x20` handshake component-info codes |
| [`data/component_inventory.csv`](data/component_inventory.csv) | Example component inventory from a capture |
| [`data/fields.json`](data/fields.json) | All of the above as one machine-readable file |

## TL;DR

- **Two BLE surfaces, one vendor tree.** Both live under Bosch's base
  `0000xxxx-eaa2-11e9-81b4-2a2ae2dbcce4`: the **diagnostic channel** `0x0011` (the rich
  `0x30` telemetry) and the officially documented **LDI** `eb21`.
- **Frame format.** A notification packs length-prefixed frames; first byte is the type
  (`0x30` telemetry, `0x10` status, `0x20` handshake, `0x60/0x70` stored-log). A telemetry
  frame is a one-field protobuf: `30 <len> <idHi> <idLo> 08 <varint>`.
- **Fields are verified, not guessed.** Motor/rider power in watts (`98-5D`/`98-5B`),
  rider torque ÷20 = N·m (`98-14`, r = 1.000 vs LDI), cadence ÷2 (`98-5A`), speed ÷100 km/h
  (`98-2D`, integrates to odometer within 0.3%), delivered energy in Wh (`80-9C`), remaining
  energy ÷10 Wh (`80-91`). Guesses like `98-15` motor torque and `80-8B` temperature are
  marked as candidates, with the exact evidence needed to close them.
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

- **Nik Leiser** — the HCI-snoop capture method and the Performance Line SX captures that
  opened the write/command channel and enabled the first cross-generation comparison.
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
