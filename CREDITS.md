# Credits

**Primary author, decode & tooling:** [redundo.app](https://redundo.app)
© 2026 redundo.app.

This is redundo.app's reverse-engineering of the Bosch Smart System BLE interface,
released publicly under CC BY 4.0 (docs & data) so anyone can use and build on it —
with attribution to redundo.app.

## With thanks to

- **Remko Weijnen — [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader)** (CC BY 4.0).
  The parallel, independent project that reads the BES3 diagnostic data over **USB-C** to the
  drive unit and publishes the ~895-address registry (`src/address-registry.json`). His work
  and ours are complementary: his maps the USB transport (where nearly everything is readable),
  this repo maps the **BLE** transport (where only a subset answers). Our full 161-field
  BLE-reachable list keys its **names and descriptions** to his registry, and it **corrected
  several of our earlier field guesses** — `98-1D` → `ROAD_SLOPE`, `80-E2` → `TOTAL_CAPACITY`
  (not wheel circumference), `98-29` → `REAR_WHEEL_CIRCUMFERENCE_USER`. His repo also includes a BLE transport, but one reconstructed
  from decompiling Bosch's Flow app and unconfirmed on hardware; this repo's BLE map is derived
  independently, from live HCI-snoop captures on a real bike. Attribution per CC BY.
- **Nik Leiser**, of the **[BikeBridge](https://codeberg.org/bg443/BikeBridge)** project — an
  open-source Android app that reads live BLE telemetry from e-bikes and sends commands, with
  transparent access to all raw data (Shimano EP8, SRAM Eagle, and reverse-engineered parsers
  for more); a kindred BLE reverse-engineering effort. Nik proposed capturing at the HCI layer
  (the Bluetooth HCI snoop log), which exposed both directions of the BLE link, and contributed
  the first Performance Line SX captures with his own ComProbe hardware sniffer — both decisive.
- **The earlier community posters** whose initial Bosch field notes gave this work its
  starting point. If you recognise your contribution and would like a direct citation,
  please open an issue.

## Cross-project confirmations

The field map was cross-checked against the open community (July 2026). No contradictions
to the verified rows; several independent confirmations. The two most central projects —
**Remko Weijnen's `bes3-reader`** (the USB registry we key every field name to) and **Nik
Leiser's BikeBridge** (the HCI-snoop method + the SX captures) — are detailed above; further
corroborations:

- **Nilogax/SmartBridge** (Android + XIAO nRF52840 → Garmin) — agrees byte-for-byte on the
  `0011` channel fields.
- **bestie-org/BEStie** (phone-free nRF52840, FTMS) — bundles the official Bosch `.proto`,
  confirming the 13 documented LDI fields.
- **RobbyPee/Bosch-Smart-System-Ebike-Garmin-Android** — first to flag `98-15` as motor
  torque; one disagreement (÷10 speed on `98-2D`) is discussed in the decoder card.
- **Xunil99/ha-bosch-ebike** (ESPHome LDI bridge) — implements the 13 official LDI fields.
- **"Nyon Unchained," arXiv 2404.12864** — forensic teardown of the Nyon BUI350 (adjacent
  work; confirms the hardware records `driverTorque`).

## Sources

- Field map + method: **redundo.app**.
- **BES3 address registry** — Remko Weijnen, `rweijnen/bosch-bes3-reader`, **CC BY 4.0**
  (github.com/rweijnen/bosch-bes3-reader). Canonical component/field names and addresses,
  reverse-engineered over USB; used here as the reference set our BLE-reachability results
  are keyed to.
- Bosch **Live Data Interface** spec V1.0 (May 2026), Apache-2.0, embedding
  `ebike_live_data.proto` — bosch-ebike.com → Business → Live Data Interface.
