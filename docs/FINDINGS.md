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

This write-up is mostly about that second, undocumented channel — what its data means, and
how we proved each field rather than guessing.

Scope is deliberately honest: **one bike, deeply verified** — a Performance Line CX (BDU3740,
PowerTube 750, fw 20.x) — cross-checked against Bosch's own Flow app, an independent reference
power meter, FIT ride logs, and (for the LDI) Bosch's official spec. It is "one model, deeply,"
never "the whole Bosch protocol."

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
- **Cadence** (`98-5A`) is raw ÷ 2 = rpm.
- **Speed** (`98-2D`, always present) is raw ÷ 100 = km/h — confirmed because integrating it
  over a whole ride reproduces the odometer to within 0.3%.
- **Delivered energy** (`80-9C`) is watt-hours; a ride's consumption is just last − first.
- **Remaining battery energy** (`80-91`) is raw ÷ 10 = Wh, and the SoC percentage (`80-88`)
  is derived from it.
- **Assist mode names, IDs, catalog, and colors** live on the `18-xx` family; the assist
  level itself is `98-09`.

And the honest not-yet-proven ones: **motor torque** (`98-15`) clearly tracks motor power
(r = 0.93) but its scaling isn't nailed down, and **`80-8B`** is temperature-*shaped* but its
absolute scale is unproven — both are marked as candidates in the card, with the exact
measurement needed to close them.

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

## 5. Cross-generation: CX vs SX

The fear that field IDs shift wholesale between generations turned out to be only partly true.
The **transport and the core power/battery IDs are identical** across the Performance Line CX
and SX. The real differences are narrow: the assist-mode config-key namespace differs
(`A100M…` on CX vs `A100E…` on SX), and Bosch's Flow app doesn't even request torque/cadence
on the SX we captured, so those didn't stream there. Treat that last point as an observed
interface difference, not proof the SX hardware lacks them.

## 6. What this does *not* show (honest limits)

- The `0x0011` stream is the drive unit's **output**, not a two-sided mirror — we never see the
  head unit *requesting* a field, because those requests run on the internal wired bus, off-BLE.
- So "torque/cadence unreachable over BLE" *leans* true but isn't fully proven.
- Wheel-circumference changes weren't located in any BLE capture; their transport is unresolved.

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
