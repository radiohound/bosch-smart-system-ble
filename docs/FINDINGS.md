# Reverse-Engineering the Bosch Smart System — a plain-English walkthrough

*A one-bike-deeply-verified study of the Bosch Smart System eBike's Bluetooth interface,
cross-checked against manufacturer ground truth and a second drive-unit generation.*

**By redundo.app.** This is the narrative "what we found and how" explainer. For the exact
field-by-field lookup table, see **[DECODER-CARD.md](DECODER-CARD.md)** — that card is the
authoritative, most-current field map, and where a detail here and there differ, the card wins.

> **Acknowledgment.** This work would not have reached the write/command interface without
> **Nik Leiser**, who suggested capturing the raw **Bluetooth HCI snoop log** — the key that
> let us see *both directions* of the BLE link rather than only the notifications an app
> subscribes to — and contributed the first captures from a **Performance Line SX** drive
> unit with his own ComProbe hardware sniffer. Both were decisive.

## The short version

A Bosch Smart System eBike talks over Bluetooth LE on two "surfaces" that live under the same
vendor service tree (base `…-eaa2-11e9-81b4-2a2ae2dbcce4`):

1. **The documented Live Data Interface (LDI)** — characteristic `eb21`. Bosch publishes a
   spec for this (13 fields: speed, cadence, rider power, SoC, odometer, a few flags).
2. **An undocumented diagnostic channel** — characteristic `0x0011`. This is the rich one,
   carrying motor power, torque, energy, assist-mode config, and much more.

There's also a **third way into the same diagnostic data that isn't Bluetooth at all**: a
**USB-C** connection to the drive unit's controller. Remko Weijnen's
[`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader) reverse-engineered that path and
published the full ~895-address BES3 registry (CC BY 4.0). The USB transport exposes *nearly
everything*; the BLE diagnostic channel exposes only a **subset** — and a large part of what
this write-up adds is exactly *which* subset. We keyed our tests to his registry and found that
of ~895 addresses, **161 answer over BLE and ~481 refuse** (the electrical internals — pack
voltage, live discharge current — stay USB-only, even under load). So: his registry is the
"what exists" map over USB; this repo is the "what you can reach over Bluetooth, and how" map.

This write-up is mostly about that second, undocumented **BLE** channel — what its data means, and
how we proved each field rather than guessing.

Scope is deliberately honest: **one bike, deeply verified** — a **smart system (gen 4) Performance
Line CX** (BDU3740, PowerTube 750, fw 20.x) — cross-checked against Bosch's own Flow app, an
independent reference power meter, FIT ride logs, and (for the LDI) Bosch's official spec. We've
also done **preliminary testing on a second bike — a Performance Line SX** (also smart system
gen 4, ~400 Wh pack, contributed by Nik): enough to confirm the core power/battery IDs and the
write grammar carry across, and to map the narrow generational differences (see §5), but not the
deep per-field verification the CX has. So it is "one model deeply, a second previewed," never
"the whole Bosch protocol."

## 1. How the data is framed

The diagnostic channel sends **length-prefixed frames**, and a single Bluetooth notification
can pack several of them back to back. The first byte of each frame says what kind it is:

| First byte | What it is |
|-----------|------------|
| `0x30` | **component telemetry** — the fields you actually want (motor power, etc.) |
| `0x10` | short periodic status |
| `0x20` | session establishment / handshake (component names, part numbers) |
| `0x60` / `0x70` | stored-log file transfer (the bike's on-board "flight recorder") |

A telemetry frame is a tiny protobuf:

```
30 <len> <idHi> <idLo> 08 <varint>       e.g.  30 .. 98 5D 08 <watts>  → motor power
```

So `<idHi><idLo>` (like `98 5D`) is the field's ID, and the number after tag `08` is its value.
To read a notification: split it on `0x30` boundaries first, then decode each frame.

## 2. What the fields mean (and how we know)

The full table with confidence markers is in the [decoder card](DECODER-CARD.md). The headline,
verified results:

- **Motor power** (`98-5D`) and **rider power** (`98-5B`) are watts, direct. Rider power matches
  the LDI's own rider-power field exactly.
- **Rider torque** (`98-14`) is raw ÷ 20 = N·m — confirmed because it comes out at exactly
  **2× the LDI torque field**, correlation r = 1.000.
- **Cadence** (`98-5A`) is raw ÷ 2 = rpm on the diagnostic channel — but the *reliable* source
  for cadence and rider torque is the **LDI** (`eb21` field 2 = cadence, field 7 = rider torque),
  which streams them every ride regardless of boot timing.
- **Wheel circumference** is the *setting itself*: `98-29` (`REAR_WHEEL_CIRCUMFERENCE_USER`) /
  `98-28` (OEM default), raw ÷ 10 = mm — on the diagnostic channel.
- **Speed** (`98-2D`, always present) is raw ÷ 100 = km/h — confirmed because integrating it
  over a whole ride reproduces the odometer to within 0.3%.
- **Delivered energy** (`80-9C`) is watt-hours; a ride's consumption is just last − first.
- **Remaining battery energy** (`80-91`) is raw ÷ 10 = Wh, and the SoC percentage (`80-88`)
  is derived from it.
- **Assist mode names, IDs, catalog, and colors** live on the `18-xx` family; the assist
  level itself is `98-09`.

Two candidates closed this round: **motor torque** (`98-15`) is raw ÷ 20 = N·m — its ride peak
lands on each bike's *rated* torque (CX **85 N·m**, SX **55 N·m**; see §5), which pins the scaling.
And **`80-8B`** battery temperature is **`zigzag(raw)/10` = °C** — it matches Bosch Flow's
`presentCellTemperature` and the registry, and reads the *internal* cell temperature, which is why
surface-IR anchors never fit it.

> ⚠️ **One trap worth repeating:** motor power (`98-5D`) is event-pushed and *omits zeros* —
> the "motor off" moments aren't sent. Don't integrate it raw for energy (you'll blow past
> 100% efficiency). Use `80-9C` for energy, or cadence-gate the motor stream first.

## 3. Catch the bike's power-on

The single most useful practical finding: **which fields you get depends on *when you
subscribe*, not on the hardware.** Subscribe *at the bike's boot* (power-cycle it with the app
already connected) and you get the full set — both speed fields, cadence, and the stored-log
transfer. Join an **already-running** bike and you get only a reduced steady-state set (no
cadence field, one speed field) for that entire power cycle. We confirmed this on one bike two
different days: the boot capture saw everything; the mid-session capture saw none of the
boot-only fields — not a decode miss, they simply weren't sent.

## 4. The write / command channel

Reading *both directions* of the link (thanks to the HCI-snoop method) showed that Bosch's own
app doesn't just listen — it **subscribes** and **writes**. A third-party bonded app of our own
could write to the channel and the bike parsed and replied, and configuration/assist-mode
writes actually took effect. The long-standing assumption that the bike ignores third-party
writes is false, and — notably — **no cryptographic handshake gates the write channel**; the
"handshake blob" some sources describe is actually a cleartext config key.

**And the write channel is also a *read* channel — this is how we display request-only fields.**
Writing a one-field request — `30 05 40 80 <idHi> <idLo> 00` — makes the bike reply on `0x0011`
with that field's value (or a "not available" status). That's what let us **map which of the
~895 registry addresses are reachable over BLE** (161 answer with data) and **pull fields that
never stream on their own**: the battery **FET** and drive-unit **PCB** temperatures
(`80-D2` / `98-84`), the remote's internal battery voltage (`A1-C1`), the battery's
max-discharge-current limit (`80-93`), and more. The request/response grammar and the full
reachability map are in **[BLE-ACCESS.md](BLE-ACCESS.md)**.

## 5. Cross-generation: CX vs SX

Field IDs are widely feared to shift wholesale between drive-unit generations — a published
report (RobbyPee issue #6: a BDU3741/CX and BDU3143/SX at FW 17.16.0) claims *none* of the
documented IDs matched. We compared a **CX hill-climb** (our BDU3740) against a **Performance
Line SX ride** (contributed by Nik) — both *moving*, so the fields line up fairly — and the
truth is far narrower: **the physics IDs and scalings are identical; only two hardware specs and
the config namespace actually differ.**

**Riding CX vs riding SX — same IDs, same scalings:**

| field (scaling) | CX (Jul climb) | SX (Nik's ride) |
|---|---|---|
| motor power `98-5D` | 6–**597 W** | 57–**333 W** |
| rider power `98-5B` | 5–296 W | 24–116 W |
| cadence `98-5A` (÷2) | 6–108 rpm | 27–70 rpm |
| rider torque `98-14` (÷20) | 5–45.5 Nm | 6.4–20.1 Nm |
| **motor torque `98-15` (÷20)** | 4.8–**85.1 Nm** | 10.7–**56.6 Nm** |
| speed `98-2D` (÷100) | 2.5–60.9 km/h | 2.5–25 km/h |
| max motor power `98-74` | 600 W | 600 W |

Every mover streams on **both** drive units' `0x0011` channel with the **same ID and scaling**.
With both captures riding there are **zero SX-only telemetry IDs** — the earlier impression that
"the SX doesn't stream torque/cadence" was a *parked-vs-riding* artifact (from a Flow capture
that simply didn't subscribe to them). The SX carries torque and cadence on the diagnostic
channel exactly like the CX.

**Full field-set cross-check.** Beyond the movers, we diffed *every* `0x30` id in a CX boot
capture against a fresh SX capture — 281 field ids shared. Crucially, **none of the SX-unique ids
are sensors**: they are all `0x20` component-inventory / handshake frames (different part numbers)
plus one state flag, `A0-51 DIAGNOSIS_PROGRAM_ACTIVE` (the SX was in diagnosis mode during
capture). No CX telemetry field is missing on the SX, and the SX exposes no telemetry field the CX
lacks — the map is the same, end to end.

**The genuine differences — two hardware specs + the namespace:**

| aspect | CX (BDU3740) | SX (Nik's) |
|--------|--------------|------------|
| **Motor torque class** (`98-15` ÷20 peak) | **85.1 → 85 Nm rated** | **56.6 → 55 Nm rated** |
| **Battery total capacity** (`80-E2` ÷100) | 2113 → **21.1 Ah** (750 Wh) | 1143 → **11.4 Ah** (~400 Wh) |
| Assist-mode config-key namespace | `A100M…` (+ `S100RUCZ20`) | `A100E…` |
| Component / firmware (Device Info `0x180A`) | head **BHU3600** / remote **BRC3600**, SW **20.27.0** | System Controller **BRC3100**, SW **20.9.0**, HW 4.1.3 |

That motor-torque row is a two-for-one: each bike's `98-15 ÷ 20` **peak lands on its exact rated
torque** (CX 85 Nm, SX 55 Nm), which *confirms the ÷20 scaling* **and** captures the real motor
difference in one shot.

> **Speed scale is ÷100 on the SX too — now positively confirmed, not just retracted.** A
> second, independent SX capture (component `BRC3100`, pack `80-E2` = 1143 → 11.4 Ah, odometer
> 1,715 km) reads `98-2D`/`98-08` at a max raw of **1043/1003 → 10.4/10.0 km/h at ÷100**; ÷10
> would be 104 km/h, physically impossible on an eBike. Every other scaling on that capture also
> matched the CX (cadence ÷2 → 78 rpm, motor torque ÷20 → 33.6 Nm under the SX's 55 rated, rider
> torque ÷20, max motor power 600 W). So the old "CX/SX speed-scaling split (÷100 vs ÷10)" is
> dead: it traced to RobbyPee's single Strava-match on **unidentified** hardware, and two verified
> SX captures now read ÷100. Both generations are ÷100.

**Bottom line:** the physics IDs and scalings carry across generations unchanged; the real
differences are **motor torque class, battery capacity, and the config namespace** — not the
telemetry map. (One CX and now **two** SX captures — re-verify on your own drive unit.)

## 6. What this does *not* show (honest limits)

- The `0x0011` stream is the drive unit's **output**, not a two-sided mirror — we never see the
  head unit *requesting* a field, because those requests run on the internal wired bus, off-BLE.
- This is **one bike, deeply** — a CX (BDU3740) — with a second-generation cross-check (SX; see
  §5). The BLE-reachability split (which addresses answer vs refuse) and some scalings may differ
  on other drive units or firmware; cross-bike captures are welcome.

## 7. Method — the capture that made it possible

The breakthrough was capturing at the **HCI layer** rather than the app layer. Android's
built-in **Bluetooth HCI snoop log** (Developer Options → set to *Full*) records every BLE
packet in both directions to a standard `btsnoop` file, retrieved with `adb bugreport`. Parsing
it (reassemble ACL → L2CAP → ATT, then split values on frame boundaries) yields the complete
bidirectional exchange — including the phone-to-bike writes an app-layer logger never sees. A
parallel path uses a ComProbe hardware sniffer, which exports the same `btsnoop` structure.

---

*Field map + method: **redundo.app**. Cross-checked against ha-bosch-ebike, Nilogax/SmartBridge,
bestie-org/BEStie, RobbyPee's decoder, and the "Nyon Unchained" teardown (arXiv 2404.12864).
See [CREDITS.md](../CREDITS.md).*
