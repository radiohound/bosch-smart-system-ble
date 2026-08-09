# BLE reachability of the BES3 registry

Remko Weijnen's [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader) documents all
~895 BES3 diagnostic addresses read over **USB-C**. This doc answers the question his (USB)
project can't: **which of those addresses are reachable over Bluetooth, and how.**

Everything here is verified on **one bike** — a Performance Line CX (BDU3740, PowerTube 750,
fw 20.x) — by requesting every readable address over the BLE diagnostic channel (`0x0011` /
write `0x0012`) and recording what came back. Data: [`../data/ble_accessibility.csv`](../data/ble_accessibility.csv),
one row per registry address, keyed to Remko's `MCSP` address + our `BLE id`.

## The headline

Of the ~895 registry addresses, requested over BLE (146 command/action addresses excluded, 5
untested):

| BLE request result | count | meaning |
|--------------------|------:|---------|
| **value**          | 161 | bike returned data |
| **supported-empty**| 102 | bike acknowledges the field but had no value at the moment (fills in under load / in the right state) |
| **not-available**  | 481 | bike **refuses** the field on the BLE interface (`40 80 10 06`) |

So only **~263 of ~895** answer with anything over BLE; **481 are USB-only.** Separately, **236
are also pushed passively** (the `passive_stream` column) — you receive those for free without
asking.

> **The parked sweep undercounts: 161 answer a request, but 182 actually report data.** These
> counts come from a *stationary* bike, so anything that only exists during a ride reads
> `supported-empty`. Mining every logged ride capture, **21 of the 102 `supported-empty` fields
> return real data while moving** — so the true "reports data" count is **161 + 21 = 182**. The 21:
> the **8 live movers** (motor/rider power, both torques, cadence, both speeds, assist mode —
> nothing to report at a standstill); the **9-field `A2-4x` activity summary** — `A2-4A/4B` avg/max
> rider power, `A2-48/49` avg/max cadence (÷2), `A2-46` avg speed (÷100), `A2-51` calories,
> `A2-43` moving time, `A2-56` trick stats, and **`A2-54 RIDER_ENERGY_SHARE`** — the bike's own
> rider-vs-motor split, validated to within **1–2%** of the integrated `98-5B`/`98-5D` energy (see
> the decoder card); plus charging-active (`80-8A`/`80-C4`), walk-assist status (`98-6A`), and the
> Kiox tiles string (`8D-23`). The `A2-4x` family auto-pushes sparsely and resets per activity —
> use the settled late-ride values.

> **The electrical internals are USB-only.** Pack cell voltage (`80-8C`), live discharge
> current (`80-94` / `80-C8`), and last-end-of-charge voltage (`80-9E`) all return
> **not-available** over BLE — and stay that way **even under load** (re-tested at up to 263 W
> motor draw; 0 of the 481 flipped). If you need those, you need Remko's USB path. The only
> voltage BLE gives you is `A1-C1` — the *remote's* internal battery (~4.185 V), not the pack.

## The BLE request/response grammar

Request a field by writing to the diagnostic write characteristic (`…0012`):

```
30 05 40 80 <idHi> <idLo> 00        e.g. 30 05 40 80 80 8C 00  → request Battery cell voltage
```

The reply arrives as a notification on `0x0011`. Two things surprise people:

1. **The reply id has the `0x80` component bit stripped.** Motor power streams as `98-5D` but
   its *reply* echoes as `18-5D`; a Battery reply for `80-8C` comes back as `00-8C`. Re-set the
   top bit (`id | 0x8000`) to match the registry.
2. **A 3-byte status word precedes the payload**, `<S> 80 <n>`:

   | reply | meaning |
   |-------|---------|
   | `C0 80 10 08 <varint>` | **supported, here's a numeric value** |
   | `C0 80 10 0A <len> <bytes>` | **supported, here's a string/blob** |
   | `C0 80 nn` (nothing after) | **supported, no value right now** |
   | `40 80 10 06` | **not available** on this interface |

   The status word is **not data** — decode only what follows it. (Parsing `c0 80 1d` as a
   protobuf field mis-reads it as a 3712 value; don't.)

## The field-delivery model

BLE fields arrive three different ways — this matters for anything that wants a *live* value:

1. **Auto-push (continuous)** — a small set (~a couple dozen) the bike streams on its own clock,
   no request needed: assist level, pack temp (`80-8B`), SoC / remaining energy (periodic),
   charging flags, and — *while moving* — motor power, speed, cadence, torque. These are free
   and live.
2. **Config-dump-once** — ~200 component/static fields (model, firmware, serials, feature
   flags) that stream **one bulk dump when a connection opens**. Not continuous; reconnect to
   get them again. (This is why a fresh connection shows ~230 fields but a mid-connection
   moment shows ~20.)
3. **Poll-on-demand (one-shot)** — the majority. A request returns **exactly one reply**, then
   silence. Verified 1:1 (1 request → 1 reply, 3 → 3, 5 → 5). To display a live value you must
   **re-request on a timer**. Example: pack temp `80-8B` is auto-pushed (free), but the
   motor-side FET/PCB temps `80-D2` / `98-84` are poll-on-demand — poll ~1 Hz (temps move
   slowly, so that's plenty).

## Scalings confirmed over BLE

- **Battery current is milliamps** (÷1000, not ÷10): `80-93` MAX_ALLOWED_DISCHARGE_CURRENT reads
  raw 60000 → **60.0 A** (a protection ceiling; a 600 W peak only pulls ~17 A of it).
- **Battery temp `80-8B`** = **zigzag(raw)/10 °C** (a plain raw/10 reads ~2× high).
- **FET `80-D2` / PCB `98-84` temps** — readable *on request* (not in the passive stream),
  same zigzag ÷10 °C.

## Method & honesty

One bike, deeply. The full-registry sweep requests each readable address once (300 ms apart,
each with a response window); the not-available set was re-tested under real load to rule out
"only appears while riding." Command/action addresses were **not** requested (a read-style
frame shouldn't poke a setter). Cross-bike captures welcome — the accessibility split may differ
by drive-unit generation and firmware.

*BLE reachability results by [redundo.app](https://redundo.app). Address registry © Remko
Weijnen, [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader), CC BY 4.0.*
