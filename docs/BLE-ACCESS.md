# BLE reachability of the BES3 registry

Remko Weijnen's [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader) documents all
~895 BES3 diagnostic addresses read over **USB-C**, and now also carries a hardware-confirmed
BLE transport of its own. This doc is the **per-address reachability map**: which of those
addresses answer over Bluetooth, *what they answer with*, and — where they refuse — the actual
refusal code.

Everything here is verified on **one bike** — a Performance Line CX (BDU3740, PowerTube 750,
fw 20.x) — by requesting every readable address over the BLE diagnostic channel (`0x0011` /
write `0x0012`) and recording what came back. Data: [`../data/ble_accessibility.csv`](../data/ble_accessibility.csv),
one row per registry address, keyed to Remko's `MCSP` address + our `BLE id`.

## The headline

**746 addresses requested, parked, 2026-08-12.** Every outcome below is derived mechanically
from the reply bytes — see [the grammar](#the-ble-requestresponse-grammar) — rather than from a
human reading a log:

| result | count | meaning |
|--------|------:|---------|
| **value**           | 171 | answered a READ with a payload |
| **supported-empty** | 80 | acknowledged the READ, no payload at that moment |
| **`DENIED`**        | 349 | exists, and the interface refuses you (`40 80 10 06`) |
| **`NO_ROUTE_FOUND`**| 89 | nothing there to answer — no route to that endpoint |
| **`UNSUPPORTED`**   | 34 | routed, but the component doesn't implement it |
| **`NOT_READY`**     | 1 | temporarily unavailable |
| **no reply at all** | 5 | requested, silence |

Separately from the READ sweep: **11 addresses answer an RPC** with data (`GET_…`/`READ_…`/
`EXECUTE_…` — callable commands, *not* readable fields), and **11 answer a SUBSCRIBE**. Those
are recorded in the CSV's `answered_as` column and deliberately do **not** count as readable.
And **236 addresses are also pushed passively** (`passive_stream`) — free, no request needed.

> **Why the CSV totals differ slightly from this table** (174 `value` / 90 `supported-empty`):
> the table is *this one capture*, while the CSV is the accumulated record. A row is upgraded
> when a capture shows more than was published, and **never downgraded on the strength of a
> single capture** — a momentary field being empty this time is not evidence it can't answer.
> The `sweep_2026_08_12` column carries this capture's verdict per address, so the two are
> always separable.

> **The four refusal codes were previously published as one bucket of 481 `not-available`.**
> That collapse hid the system's actual structure, described next.

## Two stages: policy first, then routing

A refused request has been refused by one of **two independent stages**, and which one tells you
something quite different.

The bike we tested has **one** battery, but the registry documents two (`Battery` at `80-xx`,
`Battery2` at `82-xx`) — identical field sets, one of which is not physically present. That
makes the second component a control group, and the split is total:

| `Battery2` (not fitted) | also refused on the *fitted* `Battery` |
|-------------------------|---------------------------------------:|
| its 27 `DENIED` offsets | **27 / 27** |
| its 35 `NO_ROUTE_FOUND` offsets | **0 / 35** — and 33 of those 35 *answer* on the fitted battery |

Not one offset lands on the wrong side. So the bus does this, in order:

1. **Policy, on the address, before routing.** Restricted offsets return **`DENIED`** whether or
   not anything is there to answer. A battery you do not own still refuses `82-8C`, because
   `8C` is refused on the battery you *do* own.
2. **Routing, only if policy passed.** No node registered at that address →
   **`NO_ROUTE_FOUND`**.

The offsets denied on both are exactly the electrical internals — `8C` cell voltage, `9D`, `9E`,
`9F`, `A0`–`A3`. **The privilege wall is therefore a static, address-based filter enforced at the
gateway**: not per-component, not state-dependent, not verb-dependent. That explains why nothing
shifts it — a `SUBSCRIBE` instead of a `READ`, a different source-node identity, and re-testing
under load all leave an address filter untouched. It also reframes "USB-only": those fields are
not unreachable because Bluetooth is a weaker pipe, they are on a list this gateway enforces and
the USB path attaches past it.

**Using this to detect hardware.** The two stages give a reliable presence test, provided you
only trust the right code:

| result | what it tells you about the component |
|--------|----------------------------------------|
| answers (`value` / `supported-empty`) | **present** |
| `NO_ROUTE_FOUND` | **not present** — or that component doesn't implement that feature |
| `DENIED` | **nothing at all** — a policy verdict rendered before anyone looked |

Probe presence with a non-denied offset (a serial number or part number works: `xx-01`/`xx-81`)
and the answer is unambiguous. Probe with a restricted one and you learn only that the offset is
restricted. The same logic explains the `NO_ROUTE_FOUND` rows on components you *do* have: the
LED remote (`A0-B5`…`A0-BE`, view-stripe and tile configuration) and `8D-2D` `PLAY_SOUND` on the
Kiox are features that hardware doesn't implement — no screen on the remote, no speaker on the
head unit — while the head unit's own copies of the same tile addresses (`8D-20`, `8D-2C`) answer
normally.

> **Correction (2026-08-12): 11 rows were wrong, and the cause was our own pipeline.** Fields
> including `98-08` BIKE_SPEED, `98-2D` DISPLAYED_BIKE_SPEED, `98-96` OEM_TORQUE_LIMITATION and
> both `COMPONENT_LOCK_CONFIGURATION` addresses were published as `supported-empty` but return
> real payloads. Verified two ways: our own client and `bes3-reader` return **byte-identical**
> bytes for all of them, and replies arrive in **50–137 ms**. The bug was never in the protocol,
> the tools, or the bike — it was the step that turned a capture into this CSV. That step has
> been replaced by a classifier that derives every cell from the wire bytes, so this table can
> be regenerated and audited from a capture rather than trusted.
>
> **Consequence for the rest of the map:** the `supported-empty` class is the one built by the
> old pipeline, so treat it as "not seen under these conditions" and not as evidence of absence.
> The `DENIED` rows are the sturdy ones — they are *active* replies, and `bes3-reader`'s
> independent client reproduces 213 of them exactly.

> **The 21 push-only fields below predate the fix and await a re-run while riding.** The counts
> and the field list are unchanged from the earlier ridden sweep; what is now uncertain is
> whether "returns `supported-empty` to a request even while moving" survives the corrected
> client, since parked, `98-08` *does* answer a poll (`10 01`). The passive-stream findings
> themselves are unaffected — those were mined from ride captures, not from request results.

> **Push-only fields (from the earlier ridden sweep — see the caveat above).** Running the
> sweep under load as well as parked (riding at up to ~24 km/h, ~310 W motor), **21 of the
> `supported-empty` fields returned
> `supported-empty` to a request *even while moving*** — they're **push-only**: they auto-push their
> values but never answer a poll, so a request-sweep can't see them. Mining every logged ride
> capture (up to a 206‑min ride) surfaces all 21 in the passive stream, so the true "reports data"
> count was **161 + 21 = 182**; after the 2026-08-12 re-sweep it is **174 + 19 = 193**, since
> `98-08` and `98-2D` turned out to answer a read after all. The 21 as originally identified:
> the **8 live movers** (motor/rider power, both torques, cadence, both speeds, assist mode —
> nothing to report at a standstill); the **9-field `A2-4x` activity summary** — `A2-4A/4B` avg/max
> rider power, `A2-48/49` avg/max cadence (÷2), `A2-46` avg speed (÷100), `A2-51` calories,
> `A2-43` moving time, `A2-56` trick stats, and **`A2-54 RIDER_ENERGY_SHARE`** — the bike's own
> rider-vs-motor split, validated to within **1–2%** of the integrated `98-5B`/`98-5D` energy (see
> the decoder card); plus charging-active (`80-8A`/`80-C4`), walk-assist status (`98-6A`), and the
> Kiox tiles string (`8D-23`). The `A2-4x` family auto-pushes sparsely and resets per activity —
> use the settled late-ride values.

> **The electrical internals are USB-only — refused to a READ *and* to a SUBSCRIBE.** Pack cell
> voltage (`80-8C`), live discharge current (`80-94` / `80-C8`), and last-end-of-charge voltage
> (`80-9E`) all return **`DENIED`** over BLE. They stay that way **even under load** (re-tested at
> up to 263 W motor draw; 0 of the refused set flipped), and — tested 2026-08-12 — they also
> `DENIED` a **SUBSCRIBE**, which matters because several addresses on this bus answer a
> subscribe while refusing a plain read. That possibility is now ruled out here, with a positive
> control in the same session seconds earlier: `18-57` `REACHABLE_RANGE`, which *also* refuses a
> read, subscribed successfully and pushed its value (per-assist-mode ranges 136 / 87 / 72 / 69).
> So the subscribe mechanism demonstrably works on this bike and these four fields still refuse
> it — a privilege boundary, not a wrong-verb artifact. If you need them, you need the USB path.
> The only voltage BLE gives you is `A1-C1` — the *remote's* internal battery (~4.185 V), not the
> pack.

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
   | `40 80 nn <status>` | **refused** — the last byte says *why* (below) |

   The status word is **not data** — decode only what follows it. (Parsing `c0 80 1d` as a
   protobuf field mis-reads it as a 3712 value; don't.)

   **`<n>` is `(type << 4) | seq`,** so the high nibble tells you which operation is being
   answered — `1` READ, `3` WRITE, `5` RPC, `7` SUBSCRIBE, `9` UNSUBSCRIBE. Worth decoding: an
   address that answers only an RPC is a **callable command**, not a readable field, and
   conflating the two puts commands in the map as if you could read them.

   **Refusal status bytes** (`ResponseMessageStatusCode`, per `bes3-reader`'s `protocol.js`):

   | byte | name | meaning |
   |------|------|---------|
   | `02` | `NO_ROUTE_FOUND` | nothing there to answer — no route to that endpoint |
   | `03` | `NOT_READY` | temporarily unavailable |
   | `04` | `UNSUPPORTED` | routed, but the component doesn't implement it |
   | `06` | `DENIED` | it exists, and this interface refuses you |

   Recording only "not available" throws away that distinction — which is exactly the mistake
   the earlier version of this map made.

### An argument-taking RPC reads as "not-available" under an argument-less sweep

The reachability sweep probes each address with a bare one-field request. That is the right probe
for a value, and the **wrong** probe for an RPC that expects an argument — which answers only when
it is given one.

`90-90` / `90-91` / `90-92` (the UDAM reads) and `90-8B` all sat in the `not-available` bucket for
this reason. Each of them returns data reliably when called with a `ConfigId`. Their
`ble_request_result` is now recorded as **`value (needs ConfigId arg)`**; the raw
`sweep_2026_08_12` column is left untouched, since the sweep result itself was not wrong — only the
conclusion drawn from it.

**So treat the 481 `not-available` count as an upper bound, not a verdict.** Some unknown fraction
of it is argument-takers that were never given an argument. Anything in that bucket whose registry
name reads like a command is worth re-probing properly before being written off.

### The sequence number is 4 bits — and it is shared

The low nibble of the `<SEQ>` byte is the request sequence; the high nibble is the class
(`0` read, `2` write, `4` RPC, `6` subscribe, `8` unsubscribe). That leaves **16 slots, shared by
everything in flight**, and replies carry no other way to say which request they answer.

A burst is easy to build without noticing. Reading a full mode configuration fires roughly 22
requests — a catalog read, then a name read and a parameter read per mode. The counter wraps, two
`90-8B` calls collide on the same pending key, and one reply is dropped. The symptom is a single
field that mysteriously never resolves, not an error.

**Serialise same-address reads**, or track outstanding requests by `(address, seq)` and refuse to
reuse a slot that is still open.

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
