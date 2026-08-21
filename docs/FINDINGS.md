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
published the full ~895-address BES3 registry (CC BY 4.0). His tool now reads over **Bluetooth as
well as USB**, and where the two overlap his BLE results and ours agree field-for-field — 213
shared refusals, zero disagreements — which is independent confirmation of the map below rather
than a competing one. The USB transport still exposes *nearly everything* while the BLE channel
exposes only a **subset**, and a large part of what this write-up adds is exactly *which* subset,
and why each address refuses. We keyed our tests to his registry and found that
of ~895 addresses, **178 return data to a read over BLE and 481 refuse** — 174 answer a plain
read, and 4 more (the UDAM config RPCs `90-8B`/`90-90`/`90-91`/`90-92`) return data when given a
ConfigId argument — and the refusals split four
ways (`DENIED` 348 / `NO_ROUTE_FOUND` 90 / `UNSUPPORTED` 34 / `NOT_READY` 1, with 8 more refusing
without a status recorded) rather than being one undifferentiated wall. The electrical internals — pack voltage, live discharge current — stay
USB-only, refused even under load and refused to a subscribe as well as a read — with the subscribe
verb proven working on the same bike in the same session, so that is a genuine privilege wall and not
a wrong-verb artifact. So: his registry is the "what exists" map; this repo is the "what you can reach
over Bluetooth, and why the rest refuses" map.

This write-up is mostly about that second, undocumented **BLE** channel — what its data means, and
how we proved each field rather than guessing. It also, in **§6**, follows that channel into the
**navigation display** — decoding the vector-map-tile protocol well enough to author our own tiles,
and mapping the exact cryptographic boundary that stops third parties from *displaying* them.

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
- **Speed** is raw ÷ 100 = km/h on both `98-2D` (`DISPLAYED_BIKE_SPEED`, the one always present,
  and what the head unit shows) and `98-08` (`BIKE_SPEED`). Confirmed because integrating `98-2D`
  over a whole ride reproduces the odometer to within 0.3% — and the two fields are **identical**
  here across 113,147 paired samples (median ratio 1.000), so this bike adds no optimism to the
  speed it displays, despite a `SPEED_DISPLAY_TOLERANCE` field existing in the registry.
- **Delivered energy** (`80-9C`) is watt-hours; a ride's consumption is just last − first.
- **Remaining battery energy** (`80-91`) is raw ÷ 10 = Wh, and the SoC percentage (`80-88`)
  is derived from it.
- **Assist mode names, IDs and colors** live on the `18-xx` family; the assist level itself is
  `98-09`. Two traps there: `18-0D` carries the **long** names, which matters because a bike can
  offer two modes sharing one short name (this one has both `eMTB` and `eMTB-shortcrank`), and the
  mode catalog is **split in two** — `98-68` is only the lower half, `98-0F` the upper, and a mode
  can be configured while appearing solely in the latter.

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

> **UPDATE (2026-08-21): the boot-window sensitivity is a startup HANDSHAKE, not a timing
> quirk.** The bike drives the MobileApp through a staged startup (`0x40AA` `MOBILE_APP_STATIC_FEATURE_PROPERTIES`
> read + `0x40A9` `STARTUP_STAGE` writes, marching STAGE5→STAGE9). A client that ANSWERS the
> handshake can subscribe to `0x0011` immediately and the channel stays alive — confirmed on
> hardware. Subscribing mutely during boot is what kills it. Full detail and credit in
> [STARTUP-HANDSHAKE.md](STARTUP-HANDSHAKE.md).

**But do not subscribe too *early*.** Being present at boot is what you want; firing the
notification-enable on `0x0011` in the first moment after connecting is not. Subscribe inside the
bike's boot window and **that channel stays dead for the whole power cycle** — not a slow start,
nothing recovers it — while the LDI stream on `eb21` keeps flowing happily. That combination is the
trap: it looks exactly like a successful subscribe to a bike with nothing to say.

Two things make it survivable:

- **Delay, and learn the delay.** We wait ~2 s after connect before enabling notifications on
  `0x0011`, and on a dead channel raise it a second at a time toward a 20 s ceiling, remembering the
  working value per bike. Some bikes need considerably more than 2 s.
- **Re-subscribe rather than just noting it.** Disabling and re-enabling the notification *does*
  recover the channel without a power cycle. Learning the right delay for next time is no use to
  someone staring at a dead channel now.

And judge liveness by **recency, not by a count**. A lifetime "frames received" counter stays above
zero after the channel dies, so gating on it means firing requests into a dead subscription and
concluding the bike is unresponsive.

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
~895 registry addresses are reachable over BLE** (178 return data to a read — 174 to a plain read plus 4 argument-taking UDAM RPCs) and **pull fields that
never stream on their own**: the battery **FET** and drive-unit **PCB** temperatures
(`80-D2` / `98-84`), the remote's internal battery voltage (`A1-C1`), the battery's
max-discharge-current limit (`80-93`), and more. The request/response grammar and the full
reachability map are in **[BLE-ACCESS.md](BLE-ACCESS.md)**.

**The assist-mode configuration surface is documented in full in
[MODE-CONFIG.md](MODE-CONFIG.md)** — reading and writing the four configured modes (`98-4E`), one
mode's parameters (`90-90`/`90-93`, the `UdamParams` block), the factory defaults and per-mode reset
(`90-91`/`90-94`), and the id → name resolution that `98-09` requires.

Two results from that work belong here, because they bound what this channel can do:

- **The drive unit enforces its own limits.** `90-92` caps the assist cut-off at 3200 (32.00 km/h)
  on this bike. Writing 3541 against it was refused four times and read back unchanged.
  **Derestriction is foreclosed by the protocol, not by anyone's discretion.**
- **A refusal is silent.** Both an applied and a refused write reply with the `C0` success prefix;
  only the payload differs (`08 01` = applied, empty = declined). Anything checking the prefix alone
  will report writes that never happened — judge by re-reading, never by the acknowledgement.

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

Every moving field streams on **both** drive units' `0x0011` channel with the **same ID and scaling**.
With both captures riding there are **zero SX-only telemetry IDs** — the earlier impression that
"the SX doesn't stream torque/cadence" was a *parked-vs-riding* artifact (from a Flow capture
that simply didn't subscribe to them). The SX carries torque and cadence on the diagnostic
channel exactly like the CX.

**Full field-set cross-check.** Beyond the moving fields, we diffed *every* `0x30` id in a CX boot
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

## 6. The navigation display: authoring our own map tiles — and the wall

The same `0x0011` channel carries more than telemetry. When the Flow app navigates, it **draws the
map on the Kiox display itself** — the head unit has no onboard maps; the phone streams the picture
over BLE. We set out to understand that well enough to send *our own* map, and got most of the way:
**we can author map tiles the bike accepts, byte-for-byte — but making them actually display is
gated by cryptography we can't (and shouldn't try to) forge.** Both halves are worth publishing,
the second especially. *(The full byte-level spec — frames, tile schema, coordinate math, handshake
— is in **[NAV-PROTOCOL.md](NAV-PROTOCOL.md)**; this section is the narrative.)*

**The map is vector tiles, pushed as RPC frames.** Alongside the `0x30` telemetry frames, the nav
system uses **command/RPC frames** — `30 <len> 40 80 <idHi> <idLo> <typeSeq> …` — on the same
characteristic. The one that carries the map is `8D-2B` **`SET_TILE_CONTENT`**. We decoded its
format two independent ways and they agree exactly: (a) byte-by-byte against a real Flow→Kiox HCI
capture, and (b) against Flow's **own encoder**, recovered by decompiling the app (`com.bosch.ebike`
is native Kotlin, so `jadx` reads it cleanly; the class `TileConversionKt` is the map encoder). The
model, **VERIFIED**:

```
SetTileContent{ 1: TilePosition[], 2: TileContent }
TileContent    { 1: tileIndex, 2: Layer[] }
Layer          { 1: styleIndex, 2: Shape[] }          # styleIndex → a style table sent once
Shape          { 1: RelativePoint[] }
RelativePoint  { 1: x, 2: y }                          # absolute 0..159 px in the tile; a 0 coord is omitted
```

Colors come from a **style table** the phone sends first (a list of `{ARGB color, lineWidth}`);
layers reference it by index. In practice **style 0 = white = the route** (Bosch's "trail" layer —
in trail-navigation the trail *is* your route) and **style 1 = grey = the surrounding roads**. A
tile is a `160×160` px vector image; the Kiox displays roughly one tile and pre-loads a 3×3 ring
around it for scrolling.

**The keystone — the coordinate system.** Each tile is placed by an `AbsolutePoint`, and cracking
what that means is what turns "replay Bosch's bytes" into "author our own map anywhere." It is a
**standard Web-Mercator (OSM slippy-map) tile coordinate at zoom 18, times 160** — **VERIFIED**
against the ride's GPX to a ratio of 1.0000:

```
tileX = (lon + 180)/360 · 2^18
tileY = (1 − ln(tan(lat) + sec(lat))/π)/2 · 2^18
AbsolutePoint = (tileX · 160, tileY · 160)
a road vertex's local pixel = frac(tileIndex) · 160        # 0..159 within its tile
```

So from any latitude/longitude you can compute the tile, its on-screen position, and where each
road vertex lands inside it — no lookup table, no captured coordinates.

**It's pull-based, and it accepts what we author.** The head unit drives the exchange: it
advertises which tiles it wants via `8D-23` `FEATURE_STREAMING_TILES_OF_INTEREST`, the phone
declares the feature (an `8D-26` option group), and content is only accepted for the slots the
display asked for. We built the whole set ourselves — style table, tile enumeration, position table
(from the Mercator math above), and content (a route line plus a road drawn from two real lat/lon
points) — and the bike **structurally accepts every frame**: reply `0d2b c080 … ` with **zero
rejects**, the same acceptance shape Flow's own frames get. The encoding is fully reproducible from
first principles; nothing is copied. *(One malformed-frame lesson worth recording: `SetTileContent`
puts the position/enumeration list on field 1 and the tile content on field 2 — content needs the
extra message wrapper, and omitting it is the difference between acceptance and a silent `…08`
reject.)*

**The wall — and why it's a real boundary, not a missing decode. NEGATIVE (by design).** Accepted
tiles come back marked *not displayed* (`…1000` vs Flow's `…1001`) because they only render on a
live **map canvas**, and the Kiox opens one only inside an **authenticated navigation session**.
Selecting a destination triggers a **certificate exchange** — BER-TLV `7F21` cert with `5F24/5F25`
validity dates, `5F34` sequence, `5F37` signature — i.e. a Bosch-CA-signed credential, mutually
verified (the head unit stores a `RemoteControl.publicKeyType`; the bus has a dedicated `bes3`
`Signature` message).

> **Correction (2026-08-16):** this previously read "the same gate that denies `DISPLAY_GENERIC_TEXT`
> to third parties." It is not. `8D-4A` (and `8D-24`/`8D-25`/`8D-82`) are refused by the **static
> address filter applied before routing** — verb-independent, and applied even on components the
> bike doesn't have — not by the certificate. See [`BLE-ACCESS.md`](BLE-ACCESS.md#two-stages-policy-first-then-routing).
> Note also that third parties **can** put text on a Kiox without any credential — but the two
> routes are not equal. The `C0-82` **`FeatureStreamingAlert`** renders on **every screen**, carries
> two lines, an icon and up to two labelled buttons, and is **interactive** (the bike replies with
> which button was pressed) — that is the useful surface. The option-group text page also renders
> our own content, but only on the **nav screen** and behind a "hold the button 2 s" entry gate that
> the phone **cannot** dismiss (`A0-48` and `A0-43` both refuse), so it needs the rider to open it.
> Both are detailed in [`NAV-PROTOCOL.md`](NAV-PROTOCOL.md#3a-the-c0-82-banner-is-a-full-featurestreamingalert--source-verified-from-flow).
> What no third party can do is use `8D-4A` or open a map canvas.
And decompiling Flow confirms the credential is **not extractable**: its `securestore` holds key
material in the **Android hardware Keystore** (`KeyGenParameterSpec` / TEE-StrongBox), the code
*uses* the private key but never contains it, and the trust root is **Bosch's CA** — a keypair we
generate ourselves is rejected. So the authorization is deliberately engineered to be non-forgeable,
and it works.

**Net result:** the Bosch navigation-tile protocol is fully understood and reproducible up to the
security boundary — **you can build the exact bytes of your own map; you cannot make the display
show them without Bosch's cryptographic authorization.** That negative is the useful finding: it
tells anyone chasing "custom screens on a Kiox" exactly where the line is and why it holds.

> **Note on this section vs the rest.** Everything above about the tile *format* and *coordinates*
> is verified on hardware and against Flow's own code. The auth wall is characterized from the
> handshake and the decompiled key-storage — we did not attempt to break it, and by design it
> can't be reverse-engineered around from the client alone.

## 7. What this does *not* show (honest limits)

- The `0x0011` stream is the drive unit's **output**, not a two-sided mirror — we never see the
  head unit *requesting* a field, because those requests run on the internal wired bus, off-BLE.
- This is **one bike, deeply** — a CX (BDU3740) — with a cross-check against a **second drive unit**,
  a Performance Line SX (a different product line, not a later generation of the CX; see §5). The BLE-reachability split (which addresses answer vs refuse) and some scalings may differ
  on other drive units or firmware; cross-bike captures are welcome.

## 8. Method — the capture that made it possible

The breakthrough was capturing at the **HCI layer** rather than the app layer. Android's
built-in **Bluetooth HCI snoop log** (Developer Options → set to *Full*) records every BLE
packet in both directions to a standard `btsnoop` file, retrieved with `adb bugreport`. Parsing
it (reassemble ACL → L2CAP → ATT, then split values on frame boundaries) yields the complete
bidirectional exchange — including the phone-to-bike writes an app-layer logger never sees. A
parallel path uses a ComProbe hardware sniffer, which exports the same `btsnoop` structure.

> **This method came from Nik Leiser of [BikeBridge](https://codeberg.org/bg443/BikeBridge).** He
> proposed capturing the raw HCI **snoop log** — the key that exposed *both* directions of the BLE
> link (not just the notifications an app subscribes to), which is what surfaced the phone→bike
> **write channel** — and ran the ComProbe hardware-sniffer captures, including the Performance
> Line SX. Decisive on both counts.

**For the navigation work (§6), two more methods.** Android's locked bootloader blocks HCI export on
some phones, so the Flow→Kiox nav captures were taken on an **iPhone** — Apple's Bluetooth logging
configuration profile writes a PacketLogger `.pklg` (same H4/ACL/L2CAP/ATT structure) into a
`sysdiagnose`, untethered and root-free. And Flow's **map encoder** was recovered by decompiling the
app with **`jadx`** (`com.bosch.ebike.onebikeapp` is native Kotlin, not Flutter, so it reads
cleanly) — the class `TileConversionKt` is the source of the tile model in §6, and its
`securestore`/keystore code is the source of the auth-boundary conclusion.

---

*Field map + method: **redundo.app**. Cross-checked against ha-bosch-ebike, Nilogax/SmartBridge,
bestie-org/BEStie, RobbyPee's decoder, and the "Nyon Unchained" teardown (arXiv 2404.12864).
See [CREDITS.md](../CREDITS.md).*
