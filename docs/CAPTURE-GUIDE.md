# Bosch Decoder — Capture Guide

A small Android app that reads the diagnostic BLE data stream from a **Bosch Smart System**
e‑bike (motor power, battery energy, speed, cadence, and more) and logs it to a file you can
share. It can also record a **reference power meter** (Garmin Rally / Quarq / Assioma) at the
same time, so a bike's Bosch power reading can be checked against a trusted meter.

Decode + tooling by **redundo.app**.

---

## 1. Install (sideload)

1. Get the `app-debug.apk` (shared link / GitHub release).
2. On the phone: **Settings → Apps → Special access → Install unknown apps** → allow it for the
   app you'll open the APK from (Files, Chrome, Drive…).
3. Open the `.apk`, tap **Install**.
4. First launch: grant **Nearby devices / Bluetooth** and **Location** when asked.

Requires Android **8.0+** with Bluetooth.

## 2. One‑time: pair the bike

The bike only talks to a **bonded** phone. The reliable way to bond is to **pair the bike once
in the Bosch Flow app** on this phone (turn the bike on, add it in Flow). That establishes the
OS‑level bond this decoder reuses. You only do this once.

> ⚠️ **Bond via Flow — do NOT try to make the phone an LDI *peer*.** The method above (bond in
> Flow, then connect as a **central** and reuse that OS bond) is safe. A *different* approach —
> making the phone advertise as an **LDI peripheral** so the bike connects *to it* (the
> peer/bridge path) — can **permanently burn the bond** on current gen4 firmware (CX and SX,
> post‑May FW): the bike bonds once, then **never reconnects to that phone again, and the bond
> can't be cleared** (removing the accessory in Flow doesn't help). Cause: the bike resolves a
> phone's rotating BLE MAC while *scanning* but won't use the IRK to *initiate* a connection;
> dedicated LDI computers sidestep it with a static random MAC. **This does not affect the
> central/diagnostic method in this guide** — but don't attempt the peer path on a bike you care
> about. (Reported on r/BoschSmartSystem by u/InvestigatorSenior.)

## 3. Capture a ride

> **One capturing device per bike.** The bike hands its telemetry to a *single* phone. If another
> phone/app (e.g. a recording app) is connected to the bike, this app gets only status frames and
> **no motor data**. So:

1. **Free the bike:** fully **close/force‑quit any other app** connected to it, or turn that
   phone's Bluetooth off / leave it away. If it was recently connected, **power‑cycle the bike**
   (off/on) to release its session.
2. Open Bosch Decoder → **Scan for bike** → tap your bike in the list.
3. **Wait for the green banner: "✓ On telemetry channel — 0x30 frames: N".**
   - **Green** = you're getting motor telemetry. Good to ride.
   - **Orange "⚠ STATUS‑ONLY"** = wrong channel; free the bike from other devices and
     **power‑cycle the bike**, then scan again until it turns green.
4. (Optional but valuable) In the **Reference power meter** section, connect your Rally/Quarq —
   its watts get logged alongside for the calibration check.
5. **Ride / pedal.** Logging starts automatically on connect; the field values update live.
6. When done: **Export log** → send the `.jsonl` file (Drive, email, GitHub).

Set a **session note** (e.g. "eco climb") before/while riding — it's saved in the log.

## 4. What to send

The exported `bosch-android-<timestamp>.jsonl` file. That's it — everything is in there,
including which BLE channel each frame came from. See **DECODER-CARD.md** for what the fields
mean.

## Troubleshooting

- **Bike not in the scan list** → it's held by another device or not advertising. Free it
  (above) and **power‑cycle the bike**.
- **Orange "status‑only" banner** → same fix: free the bike, power‑cycle it, rescan.
- **`MTU < 247`** (shown under the status) → fragments get truncated; disconnect and reconnect.
- **No fields at all** while pedalling → check the banner is green; if not, you're on the wrong
  channel.

---

*Questions / decode details: **redundo.app**.*
