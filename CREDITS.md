# Credits

**Primary author, decode & tooling:** [redundo.app](https://redundo.app)
© 2026 redundo.app.

This is redundo.app's reverse-engineering of the Bosch Smart System BLE interface,
released publicly under CC BY 4.0 (docs & data) so anyone can use and build on it —
with attribution to redundo.app.

## With thanks to

- **Nik Leiser** — proposed capturing at the HCI layer (the Bluetooth HCI snoop log),
  which exposed both directions of the BLE link, and contributed the first Performance
  Line SX captures with his own ComProbe hardware sniffer. Both were decisive.
- **The earlier community posters** whose initial Bosch field notes gave this work its
  starting point. If you recognise your contribution and would like a direct citation,
  please open an issue.

## Cross-project confirmations

The field map was cross-checked against the open community (July 2026). No contradictions
to the verified rows; several independent confirmations:

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
- Bosch **Live Data Interface** spec V1.0 (May 2026), Apache-2.0, embedding
  `ebike_live_data.proto` — bosch-ebike.com → Business → Live Data Interface.
