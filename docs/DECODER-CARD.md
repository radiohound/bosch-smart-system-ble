# Decoder Card — Bosch Smart System BLE format & field reference

The **format specification** and **field reference** for the Bosch Smart System's BLE data: the
on‑wire frame format, the `.jsonl` capture layout, and what every field means. It is a reference,
not a tool — any parser can implement it. Two field layers: the **verified field map** (below) —
fields byte‑confirmed with scaling and evidence — and the
**[full BLE-reachable list](#full-ble-reachable-field-list-all-161)** — **all 161** addresses that
return data over BLE, most read‑but‑not‑yet‑verified.
By **redundo.app**. Confidence markers: **✓** verified byte‑exact on a BDU3740 · **✓ᵃ**
verified byte‑exact on the Android capture (smart system BDU3740 / Performance Line CX, fw 20.x) · **?**
plausible but unconfirmed · **◐** read over BLE, scaling not yet verified · **✗** debunked (was a guess, proven wrong).

---

## Capture format

A capture is **JSONL** — one BLE notification per line:

```json
{"t": 1785035528.746, "char": "00000011", "note": "eco climb", "raw": "30049809..."}
```

| key | meaning |
|-----|---------|
| `t` | epoch **seconds** (BLE‑notification arrival). The iPhone LDI corpus uses `t_wall` = epoch **milliseconds** instead |
| `char` | which BLE characteristic it came from (full 8‑hex UUID) — **Android only**; the iPhone files are per‑channel so they omit it |
| `raw` / `raw_hex` | hex of the whole notification (may pack several frames). Android + iPhone‑diag use `raw`; the iPhone LDI corpus uses `raw_hex` |
| `note` | your session note |
| `decoded` | (iPhone LDI corpus only) the app's own field decode, for reference |
| `gps_speed_kmh` · `ctx` · `ok` | optional context — GPS speed (iPhone diag), correlation context + decode‑ok flag (iPhone LDI corpus) |

### Channels (`char`)

| char | what it is |
|------|-----------|
| `0011` | **diagnostic channel** — the `0x30` telemetry below (motor power, energy…) |
| `2A63` | **reference power meter** (standard Cycling Power) — see CPS section |
| `eb21` | **Bosch LDI** (officially documented protobuf) — see LDI section |

> The `char` value in the log is the full 8-hex UUID prefix (`00000011`, `0000EB21`, `00002A63`);
> `0011`/`eb21`/`2A63` below are shorthand. The `char` key is written by the **Android** app (all
> channels → one file); iPhone Redundo writes a **separate file per channel** instead.
>
> **Full UUIDs** (confirmed against the Nilogax/SmartBridge decoder): the diagnostic channel is
> `00000011-eaa2-11e9-81b4-2a2ae2dbcce4` and the LDI is service `eb20` / char
> `0000eb21-eaa2-11e9-81b4-2a2ae2dbcce4` — **both share the same `…-eaa2-11e9-81b4-2a2ae2dbcce4`
> vendor base**, so `0011` and `eb21` are two characteristics under Bosch's one BLE service tree.

### File layouts (three export styles)

| file | keys | channel(s) |
|------|------|-----------|
| **Android** `bosch-android-<ms>.jsonl` | `t, char, note, raw` | all — `char` tags each line |
| **iPhone diagnostic** `bosch-diag.jsonl` | `t, raw, note, gps_speed_kmh` | diagnostic `0x0011` only (no `char`) |
| **iPhone LDI corpus** `bosch-<yyyymmdd>.jsonl` | `t_wall (ms), raw_hex, decoded, ok, ctx` | LDI `eb21` only |

Each style is plain JSONL — one notification per line — so any parser can read the `raw`/`raw_hex`
hex against the field maps below. To gate motor energy on cadence when the diagnostic channel's own
cadence (`98‑5A`) wasn't in the capture (the boot‑session case below), fall back to **LDI field 2**.

## Frame types (first byte of `raw`)

A notification can pack several frames. The first byte is the frame type:

| type | meaning |
|------|---------|
| **`0x30`** | **component telemetry** — the fields you want (motor power etc.) |
| `0x40` | additional telemetry frames (seen alongside `0x30`; not yet fully decoded) |
| `0x10` | short periodic status |
| `0x20` | session establishment / handshake |
| `0x60` / `0x70` | stored‑log file transfer → the `.bin` set (index + per‑component logs) — **see "Stored‑log files" below** | 

If a capture has only `0x10` and no `0x30`, it was on a **status‑only** connection (another
device held the telemetry session) — no motor data in it.

## Decoding a `0x30` telemetry frame

```
30 <len> <idHi> <idLo> 08 <varint>
```

- `<idHi><idLo>` = the message id (e.g. `98 5D`).
- Payload is a one‑field protobuf: tag `08`, then a base‑128 **varint** = the value.
- Text fields use tag `0A` + length + string instead of `08` + varint.
- Split the notification on `0x30` boundaries first (each frame is `30 len …`, `len` bytes of body).

### Verified field map

| id | field | scaling | conf |
|----|-------|---------|:---:|
| `0x98‑5D` | **Motor power** | watts (direct) | ✓ |
| `0x98‑5B` | **Rider power** | watts (direct) — matches LDI field 5 | ✓ |
| `0x98‑14` | **Rider torque** | raw **÷ 20** = Nm — cross‑checked against LDI field 7 at **r = 1.000** (98‑14 = exactly 2× LDI‑7) | ✓ᵃ |
| `0x98‑15` | **Motor torque** | raw **÷ 20** = Nm — **verified across two bikes**: the ride peak lands on each motor's *rated* torque (CX 85.1 → 85 Nm; SX 56.6 → 55 Nm). Also r = 0.93 with motor power. `DriveUnit.MOTOR_TORQUE` | ✓ |
| `0x98‑1D` | **Road slope** | small ints 1–62, weak (r≈0.5) motor‑power correlation — which fits **gradient** (steeper → more assist), not the earlier "motor current" guess. Matches **`DriveUnit.ROAD_SLOPE`** in the bes3‑reader registry; exact scaling not yet pinned | ? |
| `0x80‑9C` | **Delivered energy** | Wh (direct, monotonic; **ride Wh = last − first**) | ✓ |
| `0x80‑88` | Battery SoC | % | ✓ |
| `0x98‑5A` | Cadence | raw **÷ 2** = rpm. Present **only when the capture catches the bike's boot session** (see capture note); otherwise absent — gate on **LDI field 2** instead | ✓ |
| `0x98‑08` | Speed (2nd field) | raw **÷ 100** = km/h — same speed as `98-2D`, appears **alongside** it when the boot session is caught | ✓ |
| `0x98‑2D` | **Speed (always present)** | raw **÷ 100** = km/h — verified integrates to odometer within **0.3%** (−0.2% measured) | ✓ᵃ |
| `0x98‑09` | Assist level | 0–4 | ✓ |
| `0x98‑18` | Odometer | metres — matches LDI field 12 | ✓ |
| `0x80‑E2` | **Battery total capacity** | raw **÷ 100** = Ah (2113 → **21.1 Ah** on the PowerTube 750; Nik's SX reads 1143 → 11.4 Ah) — matches **`Battery.TOTAL_CAPACITY`** in the registry. **NOT wheel circumference** (the earlier guess): it never changes with the Flow wheel setting because it isn't wheel‑related | ✓ |
| `0x98‑29` | **Rear wheel circumference (user)** | raw **÷ 10** = mm — the Flow wheel setting itself (`0x98‑28` = OEM default; `0x98‑2F` = allowed limits). `DriveUnit.REAR_WHEEL_CIRCUMFERENCE_USER`. This is where wheel circumference actually lives (the earlier `80-E2` guess was battery capacity) | ✓ |
| `0x98‑0C` | Assist mode names | **absent on smart system** — names arrive on `18-0D` instead | ✗ᵃ |
| `0x18‑0D` | **Assist mode names** (configured) | string list, index = `98-09` value (e.g. OFF/ECO/TOUR+/eMTB‑shortcrank/eMTB+) | ✓ᵃ |
| `0x98‑4E` | Assist mode **IDs** (configured slots) | string list (e.g. A100M00040, A100MSPIC7…) | ✓ᵃ |
| `0x18‑68` | Assist mode **catalog** | all available modes, id + name pairs (ECO/ECO+/TOUR/TOUR+/SPORT/eMTB/eMTB+/TURBO/AUTO…) | ✓ᵃ |
| `0x18‑0E` | **Per‑mode display colors** | one RGBA varint per configured mode, same order as `18-0D` (OFF = transparent; e.g. ECO `#78BE20`, TOUR+ `#00A5D8`, eMTB `#9643ED`, TURBO‑red `#E20015` — the Kiox/LED‑Remote mode colors) | ✓ᵃ |
| `0x00‑9B` | **Battery name** | text ("PowerTube 750"); uses extended addressing `C0 80 10` before the `0A` tag | ✓ |
| `0x98‑74` | Motor ratings | two varints, W (250/600 on BDU3740; 600/600 on Android capture) | ✓ᵃ |
| `0xA1‑86` | Product string | text ("smart system eBike") | ✓ᵃ |
| `0xA1‑81` | Locale | text (e.g. "en-GB") | ✓ᵃ |
| `0x80‑8B` | **Battery pack temperature** | **°C = zigzag(raw)/10** (raw is a plain varint; a bare raw/10 reads ~2× high — zigzag‑decode first, `zigzag(n) = (n>>1) ^ -(n&1)`). Matches Bosch Flow's `presentCellTemperature` and the registry (`Battery.PRESENT_PACK_TEMPERATURE`, signed ÷10); a 60‑min charge test tracks a textbook 22.5 → 28.6 °C warm‑up. This is the **internal cell** temp, so **surface IR anchors don't apply** — the earlier surface‑IR linear fit (°C ≈ 0.35·raw − 139) is superseded for that reason | ✓ |
| `0x80‑D2` | **Battery FET temperature** | **request‑only** (not passively streamed) — reads a plausible ~28–35 °C at `zigzag(raw)/10` (the registry's signed‑÷10). `Battery.PRESENT_FET_TEMPERATURE` | ? |
| `0x98‑84` | **Drive‑unit PCB temperature** | **request‑only** — reads ~23–27 °C at `zigzag(raw)/10`, warmer under load. `DriveUnit.PRESENT_PCB_TEMPERATURE` | ? |
| `0xA1‑C1` | **Remote internal battery** voltage | ~**4.185 V** (÷1000) — the remote/head‑unit's *own* cell, **not** the traction pack. `RemoteControl.INTERNAL_BATTERY_VOLTAGE`. The **only voltage readable over BLE** (pack voltage `80-8C` is USB‑only) | ? |
| `0x80‑93` | **Max allowed discharge current** | raw is **milliamps** → **÷ 1000** = A: 60000 → **60.0 A** — a protection ceiling (a 600 W peak pulls only ~17 A). `Battery.MAXIMUM_ALLOWED_DISCHARGE_CURRENT`. Live current `80-94` is **USB‑only, refused over BLE even under load** | ✓ᵃ |
| `0x98‑57` | **Per‑mode range estimates** | one byte per configured mode, km (e.g. `27 1f 14 10` = 39/31/20/16 km); identical to LDI field 3; recomputes with riding style, declines with SoC | ✓ᵃ |
| `0xA2‑43` | Timer / uptime counter | **NOT temperature** | ✗ |
| `0x80‑91` | **Remaining battery energy** | ÷ 10 = Wh. `80-88` SoC is derived from it: implied full capacity constant at **724 Wh** across 15 checkpoints (78→66 %) and the 58 % capture. Battery‑out vs `80-9C` delivered ≈ 91 % | ✓ᵃ |
| `0x80‑92` | = `80-91` **+ 50, always** (constant 5 Wh offset) | — | ✓ᵃ |
| `0x80‑C5` | exact duplicate of `80-91` | — | ✓ᵃ |
| `0x10‑90…92` | Per‑mode tune parameters | support %, max torque (40/85 Nm), max power (600 W)… | ? |

> **Capture note — catch the bike's power-on.** Which fields you get depends on **when you
> subscribe**, not on hardware or firmware. Subscribe **at the bike's boot** (power-cycle the bike
> with the app already connected) and you get the FULL set: both speed fields (`98-08` + `98-2D`),
> cadence (`98-5A`), and the `0x60`/`0x70` stored-log transfer. Join an **already-running** bike and
> you get only a reduced steady-state set — `98-2D` speed, motor, energy, temperature — with **no
> `98-08`/`98-5A`**, for that whole power cycle. Confirmed on one BDU3740 (smart system fw 20.x): an
> iPhone capture that caught the boot session saw all three (98-08 ×1377, 98-5A ×330, plus the file
> transfer); an Android grab of the **same bike** one day later, joining mid-session, saw none of
> them (0 in the raw — not a decode miss). When a mid-session capture lacks `98-5A`, cadence-gate on
> **LDI field 2** instead.

### Handshake inventory (`0x20`‑prefix frames)

During session establishment the `0011` channel also answers with component info frames
(body starts `20 xx C0 80 …`, text payload after tag `0A`): type code (`20-62`, e.g.
"EB1310000E" — the battery's **part/type code**, not its name; the friendly name is on `00-9B`),
part number (`20-61`), component model (`20-65`), manufacture date (`20-66`), firmware versions
(`20-63/64/6B/6E`), and accessory names (`20-6F`). One block per component — the Android capture
inventories: drive unit **BDU3740 "Performance Line CX"**, battery **BBP3775 "PowerTube 750"**,
head unit **BHU3600 "Kiox 300"**, remote **BRC3600 "LED Remote"**, brand "Cannondale", and the
region config "20mph_US-CA-NZ".

### Stored‑log files (`0x60`/`0x70` transfer → the `.bin` set)

When a capture **catches the bike's boot session** the `0x60`/`0x70` file transfer runs, and Redundo
drops a set of `.bin` files **next to** the `.jsonl` capture (the "untitled folder" export is the
worked example: `bosch-diag.jsonl` + `bosch-20260725.jsonl` + three `.bin`). They are the bike's
on‑board flight recorder — the same logs a Bosch service tool downloads — as raw protobuf.

**The index — `foldercontent.bin`.** A protobuf directory listing, one entry per stored file:

```
f1 { f1 { f1: "14342_1784930165.bin" }  f2: 34811 }   ← f1.f1.f1 = filename, f2 = size (bytes)
f1 { f1 { f1: "14346_1784930164.bin" }  f2: 515   }
```

Filename = `<logId>_<epoch>.bin`; the epoch is the **dump time** (1784930165 = 2026‑07‑24 21:56 UTC),
not the ride time. One `.bin` per component (`14342` = the BDU3740 drive unit, `14346` = the BRC3600
LED Remote).

**Each component log** = a repeated‑record protobuf: **record 0 is an identity header**, records
1…N are a periodic telemetry time‑series.

```
record 0 (header): f1{ f1:<serial/uid>  f2:"20.27.0"(fw)  f3:"BDU3740"(model)
                       f7:"27327-1710-42-300-00-0000"(part no)  f8:"20.9.0"(fw2) }
record 1…1050:     f2{ f1:<monotonic counter>  f2{ f1,f2, f4{f1,f3} }  f3  f4  f5 }
```

The BDU3740 log holds **1050 telemetry records** in 34 KB. The header fields decode cleanly (they
match the `0x20` handshake inventory); the per‑record `f1…f5` values are a **usage histogram / trip
recorder and are not yet field‑mapped** — decoding them is the natural next project, and needs a log
pulled right after a ride of known profile to anchor the counters.

> ⚠️ **Motor power (`0x98‑5D`) is event‑pushed and OMITS ZEROS** — the motor‑off state isn't
> sent. **Do not integrate it raw** for energy (you'll exceed 100% efficiency). Get energy from
> `0x80‑9C`. If you need mechanical work, **cadence‑gate** the motor stream: treat motor = 0
> whenever cadence = 0 (Bosch is pedal‑assist only), then integrate. Cadence source: `0x98‑5A`
> when the boot session was caught, otherwise **LDI field 2** (always available).

## Full BLE-reachable field list (all 161)

Every address that **returned data** to a BLE request on our BDU3740 — the addresses Bosch's own
**Flow app** reads, reverse-engineered, and confirmed by us **actually pulling a value back over
Bluetooth**. They are the **161 of Remko's ~895** that answer over BLE; the rest are USB-only or
command-type (full sweep: [`data/ble_accessibility.csv`](../data/ble_accessibility.csv), grammar in
[`BLE-ACCESS.md`](BLE-ACCESS.md)).

**Confidence:** **✓** = scaling + meaning byte-verified by us (full detail in the map above) · **◐**
= **read, not yet verified** — the bike returns a value and the registry names it, but we haven't
byte-confirmed the scaling on *this* bike. **Unit / scaling** = our pinned conversion where we have
one (e.g. `zigzag/10 °C`, `÷100 Ah`); a bare unit or `—` = raw. **Description** is Remko Weijnen's
`bes3-reader` registry description (CC BY 4.0). **Stream:** *auto* = pushed passively · *req* =
request-only.

*19 verified · 142 read-but-unverified · 161 reachable total.*

### DriveUnit (60)

| id | field | description | unit / scaling | stream | conf |
|----|-------|-------------|----------------|:------:|:----:|
| `90‑9A` | ALL_ISSUE_VISUALIZATION_EVENTS | — | — | auto | ◐ |
| `98‑01` | SERIAL_NUMBER | — | — | auto | ◐ |
| `98‑02` | PART_NUMBER | — | — | auto | ◐ |
| `98‑03` | PRODUCT_CODE | — | — | auto | ◐ |
| `98‑04` | HARDWARE_VERSION | — | — | auto | ◐ |
| `98‑05` | HARDWARE_SOFTWARE_VERSION | HW/SW Version | — | auto | ◐ |
| `98‑06` | SOFTWARE_VERSION | SW Version | — | auto | ◐ |
| `98‑07` | BOOTLOADER_SOFTWARE_VERSION | FBL Version | — | auto | ◐ |
| `98‑0C` | ASSIST_MODE_SHORT_NAMES | — | n/a | auto | ◐ |
| `98‑0D` | ASSIST_MODE_LONG_NAMES | — | string list | auto | ✓ |
| `98‑0E` | ASSIST_MODE_COLORS | — | RGBA varint per mode | auto | ✓ |
| `98‑0F` | AVAILABLE_ASSIST_MODES_UPPER | — | — | auto | ◐ |
| `98‑11` | DISTRACTED_RIDING_ALERT | — | — | auto | ◐ |
| `98‑12` | DRIVE_UNIT_FEATURE_PROPERTIES_RELEASE1 | — | — | auto | ◐ |
| `98‑13` | MAXIMUM_LEGAL_BIKE_SPEED | — | km/h | req | ◐ |
| `98‑17` | MAXIMUM_ASSISTANCE_SPEED | — | km/h | auto | ◐ |
| `98‑18` | ODOMETER | — | metres | auto | ✓ |
| `98‑19` | POWER_ON_TIME | Power-On Time | s | auto | ◐ |
| `98‑1A` | BIKE_NOT_DRIVING | Bike detected as not driving (motor cut-off state) | — | auto | ◐ |
| `98‑1D` | ROAD_SLOPE | — | raw (gradient) | auto | ◐ |
| `98‑25` | WALK_ASSIST_CONFIGURATION_OEM | — | — | auto | ◐ |
| `98‑26` | WALK_ASSIST_CONFIGURATION | — | — | auto | ◐ |
| `98‑27` | PRODUCT_LINE | — | — | auto | ◐ |
| `98‑28` | REAR_WHEEL_CIRCUMFERENCE_OEM | Rear Wheel Circumference (OEM) | ÷10 mm (OEM default) | auto | ◐ |
| `98‑29` | REAR_WHEEL_CIRCUMFERENCE_USER | Rear Wheel Circumference (User) | ÷10 mm | auto | ✓ |
| `98‑2A` | OEM_BRAND_IDENTIFIER | — | — | auto | ◐ |
| `98‑2B` | GEARING_SYSTEM | — | — | auto | ◐ |
| `98‑2C` | BIKE_ID | eBike ID | — | auto | ◐ |
| `98‑2E` | PRODUCT_NAME | — | — | auto | ◐ |
| `98‑2F` | REAR_WHEEL_CIRCUMFERENCE_USER_LIMITS | — | — | auto | ◐ |
| `98‑37` | MANUFACTURING_DATE | — | — | auto | ◐ |
| `98‑38` | PRODUCT_APPLICATION_REQUIRED | — | — | auto | ◐ |
| `98‑39` | PRODUCT_APPLICATION_AVAILABLE | — | — | auto | ◐ |
| `98‑3A` | GEARSHIFT_APPLICATION_REQUIRED | — | — | auto | ◐ |
| `98‑3B` | GEARSHIFT_APPLICATION_AVAILABLE | — | — | auto | ◐ |
| `98‑3C` | REGIO_SPEED_APPLICATION_REQUIRED | — | — | auto | ◐ |
| `98‑3D` | REGIO_SPEED_APPLICATION_AVAILABLE | — | — | auto | ◐ |
| `98‑42` | MAXIMUM_ASSISTANCE_SPEED_IBD | Maximum Assistance Speed (IBD) | km/h | auto | ◐ |
| `98‑46` | UDAM_MODIFICATION_POSSIBLE | — | — | auto | ◐ |
| `98‑4E` | ACTIVE_ASSIST_MODES | — | string list | auto | ✓ |
| `98‑55` | BIKE_CATEGORY | — | — | auto | ◐ |
| `98‑57` | REACHABLE_RANGE | Reachable Range (per mode, km) | one byte/mode (km) | auto | ✓ |
| `98‑5E` | OEM_BIKE_ID | — | — | auto | ◐ |
| `98‑5F` | OEM_MANUFACTURING_LOCATION | — | — | auto | ◐ |
| `98‑61` | OEM_MANUFACTURING_DATE | — | — | auto | ◐ |
| `98‑65` | BIKE_NOT_MOVING | Bike detected as stationary | — | auto | ◐ |
| `98‑68` | AVAILABLE_ASSIST_MODES_LOWER | — | id+name pairs | req | ✓ |
| `98‑69` | REQUIRED_ASSIST_MODES_LOWER | — | — | auto | ◐ |
| `98‑6C` | OEM_BRAND_NAME | — | — | auto | ◐ |
| `98‑6D` | POWER_ON_TIME_WITH_MOTOR_SUPPORT | Power-On Time (Motor Support) | s | auto | ◐ |
| `98‑70` | DRIVE_UNIT_FEATURE_PROPERTIES_RELEASE2 | — | — | auto | ◐ |
| `98‑71` | DRIVE_UNIT_FEATURE_PROPERTIES_RELEASE3 | — | — | auto | ◐ |
| `98‑74` | MAXIMUM_AVAILABLE_MOTOR_POWER | — | two varints (W) | auto | ✓ |
| `98‑75` | OEM_BIKE_MODEL_ID | — | — | auto | ◐ |
| `98‑7A` | DRIVE_UNIT_FEATURE_PROPERTIES_RELEASE4 | Feature-properties bitset, release 4 | — | auto | ◐ |
| `98‑7D` | REGIO_SPEED_CONFIGURATION | Regional Speed Configuration ("Speed ID") | — | auto | ◐ |
| `98‑80` | DRIVE_UNIT_STATIC_FEATURE_PROPERTIES | — | — | auto | ◐ |
| `98‑84` | PRESENT_PCB_TEMPERATURE | — | zigzag/10°C (registry) | req | ◐ |
| `98‑8B` | ASSIST_MODE_LIMITS | Per-mode assist limits | — | auto | ◐ |
| `98‑8C` | MAXIMUM_CONFIGURED_DISCHARGE_CURRENT | Configured max battery discharge current | A | auto | ◐ |

### Battery (27)

| id | field | description | unit / scaling | stream | conf |
|----|-------|-------------|----------------|:------:|:----:|
| `80‑81` | SERIAL_NUMBER | — | — | auto | ◐ |
| `80‑82` | PART_NUMBER | — | — | auto | ◐ |
| `80‑83` | PRODUCT_CODE | — | — | auto | ◐ |
| `80‑84` | HARDWARE_VERSION | — | — | auto | ◐ |
| `80‑85` | HARDWARE_SOFTWARE_VERSION | — | — | auto | ◐ |
| `80‑86` | SOFTWARE_VERSION | SW Version | — | auto | ◐ |
| `80‑87` | BOOTLOADER_SOFTWARE_VERSION | FBL Version | — | auto | ◐ |
| `80‑88` | STATE_OF_CHARGE | — | % | auto | ✓ |
| `80‑8B` | PRESENT_PACK_TEMPERATURE | — | zigzag/10°C | auto | ✓ |
| `80‑91` | REMAINING_ENERGY_FOR_RIDER | Remaining Energy (Rider) | ÷10 Wh | auto | ✓ |
| `80‑92` | REMAINING_ENERGY | — | constant 5 Wh offset | auto | ✓ |
| `80‑93` | MAXIMUM_ALLOWED_DISCHARGE_CURRENT | — | ÷1000 A (mA) | req | ✓ |
| `80‑96` | NUMBER_OF_FULL_CHARGE_CYCLES | Full Charge Cycles | — | auto | ◐ |
| `80‑9B` | PRODUCT_NAME | — | text | auto | ✓ |
| `80‑9C` | DELIVERED_WH_OVER_LIFETIME | Delivered Wh (Lifetime) | Wh (direct monotonic) | auto | ✓ |
| `80‑A4` | MANUFACTURING_DATE | — | — | auto | ◐ |
| `80‑B0` | NUMBER_OF_FULL_CHARGE_CYCLES_ON_BIKE | Full charge cycles counted on-bike | cycles | auto | ◐ |
| `80‑B4` | TOTAL_ENERGY | Total energy delivered | Wh | auto | ◐ |
| `80‑BC` | SYSTEM_STATE_OF_CHARGE_FOR_RIDER | — | % | auto | ◐ |
| `80‑C2` | FEATURE_PROPERTIES_RELEASE4 | Feature properties, release 4 | — | auto | ◐ |
| `80‑C5` | INSTANCE_REMAINING_ENERGY_FOR_RIDER | Remaining energy for rider (instance) | n/a | auto | ✓ |
| `80‑CA` | INSTANCE_STATE_OF_CHARGE_FOR_RIDER | State of charge for rider (instance) | % | auto | ◐ |
| `80‑D2` | PRESENT_FET_TEMPERATURE | — | zigzag/10°C (registry) | req | ◐ |
| `80‑D7` | DELIVERED_AH_OVER_LIFETIME | Delivered Ah (Lifetime) | Ah | auto | ◐ |
| `80‑D8` | STATE_OF_HEALTH | — | % | auto | ◐ |
| `80‑D9` | SYSTEM_TOTAL_ENERGY_FOR_RIDER | — | Wh | auto | ◐ |
| `80‑E2` | TOTAL_CAPACITY | — | ÷100 Ah | auto | ✓ |

### RemoteControl (56)

| id | field | description | unit / scaling | stream | conf |
|----|-------|-------------|----------------|:------:|:----:|
| `A0‑11` | ACTIVE_UI_PRIORITY | — | — | auto | ◐ |
| `A0‑21` | TIME_ZONE | — | — | req | ◐ |
| `A0‑22` | TIME | — | — | req | ◐ |
| `A0‑23` | LOCAL_TIME_OFFSET | — | — | req | ◐ |
| `A0‑25` | POWER_CYCLE_TIME | — | — | req | ◐ |
| `A0‑41` | AVAILABLE_BUTTONS | Available buttons bitmask | — | auto | ◐ |
| `A0‑61` | SERIAL_NUMBER | — | — | auto | ◐ |
| `A0‑62` | PART_NUMBER | — | — | auto | ◐ |
| `A0‑63` | HARDWARE_VERSION | — | — | auto | ◐ |
| `A0‑64` | HARDWARE_SOFTWARE_VERSION | — | — | auto | ◐ |
| `A0‑65` | PRODUCT_CODE | — | — | auto | ◐ |
| `A0‑66` | MANUFACTURING_DATE | — | — | auto | ◐ |
| `A0‑6B` | SOFTWARE_VERSION | — | — | auto | ◐ |
| `A0‑6E` | BOOTLOADER_SOFTWARE_VERSION | — | — | auto | ◐ |
| `A0‑6F` | PRODUCT_NAME | — | — | auto | ◐ |
| `A0‑81` | DRIVE_UNIT_AVAILABLE | — | — | auto | ◐ |
| `A0‑83` | BATTERY1_AVAILABLE | — | — | auto | ◐ |
| `A0‑85` | HEAD_UNIT_AVAILABLE | — | — | auto | ◐ |
| `A0‑C6` | SOFTWARE_UPDATE_DOWNLOADS_FINISHED | — | — | req | ◐ |
| `A0‑C7` | CURRENT_MANIFEST | — | — | auto | ◐ |
| `A0‑E6` | DRIVE_UNIT_UDS_IDENTIFICATION_DATA | — | — | req | ◐ |
| `A0‑E7` | BATTERY1_UDS_IDENTIFICATION_DATA | — | — | req | ◐ |
| `A0‑E8` | HEAD_UNIT_UDS_IDENTIFICATION_DATA | — | — | req | ◐ |
| `A0‑F1` | BOOTLOADER_ERROR_STATES | — | — | req | ◐ |
| `A0‑F4` | STORED_SOFTWARE_UPDATE_STATUS | — | — | auto | ◐ |
| `A0‑F5` | STORED_BOOTLOADER_ERROR_STATES | — | — | req | ◐ |
| `A1‑01` | REMOTE_CONTROL_FEATURE_PROPERTIES_RELEASE1 | — | — | auto | ◐ |
| `A1‑08` | REMOTE_CONTROL_FEATURE_PROPERTIES_RELEASE2 | — | — | auto | ◐ |
| `A1‑09` | REMOTE_CONTROL_FEATURE_PROPERTIES_RELEASE3 | — | — | auto | ◐ |
| `A1‑0A` | REMOTE_CONTROL_FEATURE_PROPERTIES_RELEASE4 | — | — | auto | ◐ |
| `A1‑0B` | REMOTE_CONTROL_STATIC_FEATURE_PROPERTIES | — | — | auto | ◐ |
| `A1‑41` | AMBIENT_BRIGHTNESS | — | — | req | ◐ |
| `A1‑51` | ACTIVE_ISSUE_VISUALIZATION_EVENT | — | — | auto | ◐ |
| `A1‑62` | GEAR_CADENCE_LIMITS | — | — | auto | ◐ |
| `A1‑65` | CURRENT_GEAR_CADENCE_SETPOINT | — | — | auto | ◐ |
| `A1‑81` | LANGUAGE | — | text | auto | ✓ |
| `A1‑82` | UNITS | — | — | auto | ◐ |
| `A1‑83` | TIME_FORMAT | — | — | auto | ◐ |
| `A1‑84` | BIKE_ID | — | — | req | ◐ |
| `A1‑86` | BIKE_NAME | — | text | auto | ✓ |
| `A1‑98` | MAXIMUM_BATTERIES_AVAILABLE_AT_SOME_POINT | — | — | req | ◐ |
| `A1‑9B` | LOCK_SOUND_ENABLED | Lock/alarm sound enabled setting | — | auto | ◐ |
| `A1‑C1` | INTERNAL_BATTERY_VOLTAGE | — | ÷1000 V | req | ◐ |
| `A1‑C5` | INTERNAL_BATTERY_TEMPERATURE | — | — | req | ◐ |
| `A1‑C6` | PCB_TEMPERATURE | — | — | req | ◐ |
| `A2‑0F` | BLE_CENTRAL_GET_AVAILABLE_DATABASE_SLOTS | — | — | req | ◐ |
| `A2‑10` | BLE_CENTRAL_DATABASE | — | — | auto | ◐ |
| `A2‑16` | BLE_CENTRAL_SCAN_FILTER | — | — | req | ◐ |
| `A2‑41` | ACTIVITY_ID | Ride/activity session identifier | — | auto | ◐ |
| `A2‑44` | TIME_ZONE_OF_ACTIVITY | Timezone/UTC offset at activity start | min | auto | ◐ |
| `A2‑4C` | AVERAGE_HEART_RATE | Average heart rate of ride activity | bpm | auto | ◐ |
| `A2‑4D` | MAXIMUM_HEART_RATE | Maximum heart rate of ride activity | bpm | auto | ◐ |
| `A2‑50` | AUTOMATIC_ACTIVITY_RESET | Automatic activity-reset trigger condition | — | auto | ◐ |
| `A2‑52` | ASSIST_MODE_USAGE_TOTAL | Per-assist-mode total usage duration | — | auto | ◐ |
| `A2‑53` | ASSIST_MODE_USAGE_WITH_MOTOR_SUPPORT_ACTIVE | Per-assist-mode usage duration while motor actively assisting | — | req | ◐ |
| `A2‑55` | BRAKE_EVENTS | Aggregate brake-event counters | — | auto | ◐ |

### HeadUnit (18)

| id | field | description | unit / scaling | stream | conf |
|----|-------|-------------|----------------|:------:|:----:|
| `8D‑01` | SERIAL_NUMBER | — | — | auto | ◐ |
| `8D‑02` | PART_NUMBER | — | — | auto | ◐ |
| `8D‑03` | PRODUCT_CODE | — | — | auto | ◐ |
| `8D‑04` | HARDWARE_VERSION | — | — | auto | ◐ |
| `8D‑05` | HARDWARE_SOFTWARE_VERSION | — | — | auto | ◐ |
| `8D‑06` | SOFTWARE_VERSION | — | — | auto | ◐ |
| `8D‑07` | BOOTLOADER_SOFTWARE_VERSION | — | — | auto | ◐ |
| `8D‑08` | MANUFACTURING_DATE | — | — | auto | ◐ |
| `8D‑09` | PRODUCT_NAME | — | — | auto | ◐ |
| `8D‑19` | HEAD_UNIT_FEATURE_PROPERTIES_RELEASE2 | — | — | auto | ◐ |
| `8D‑1B` | KEY_DEVICE | Status of phone-as-key-device pairing | — | auto | ◐ |
| `8D‑1D` | HEAD_UNIT_FEATURE_PROPERTIES_RELEASE3 | — | — | auto | ◐ |
| `8D‑1E` | HEAD_UNIT_FEATURE_PROPERTIES_RELEASE4 | — | — | auto | ◐ |
| `8D‑1F` | UNLOCK_TOKENS_NONCE | Freshness/anti-replay nonce for unlock-token generation | — | auto | ◐ |
| `8D‑22` | VIEW_STRIPE_CONFIGURATION | — | — | auto | ◐ |
| `8D‑84` | HEAD_UNIT_STATIC_FEATURE_PROPERTIES | — | — | auto | ◐ |
| `8D‑85` | SUPPORTED_TILE_IDS | — | — | req | ◐ |
| `8D‑8A` | SUPPORTED_TILE_SIZES | — | — | req | ◐ |

## Bosch LDI (`char` = `eb21`)

**Officially documented** by Bosch: *Live Data Interface* spec **V1.0** (May 2026), free,
Apache‑2.0, no registration — bosch-ebike.com → Business → Live Data Interface. Requires
smart system control‑unit firmware **v19+**.

- Service `eb20`, characteristic `eb21`, UUID base `0000xxxx-eaa2-11e9-81b4-2a2ae2dbcce4`.
- Pairing: LE Secure Connections with mandatory bonding (smartphone support out of scope).
- A GATT **read** returns a full snapshot of all values (§2.2.3.2); **notifications** carry
  changes and **may omit unchanged fields** — absence ≠ stale (§2.2.4.3).
- Payload: one proto3 message, all documented fields varint.
- **Read the proto straight from Bosch:** the official spec PDF embeds `ebike_live_data.proto`
  (`package com.bosch.ebike; message LiveData`, Apache‑2.0). It defines **exactly** fields
  1,2,5,9,10,11,12,17,21,22,23,24,25 — i.e. the 13 documented rows below and **nothing else**.
  Our fields 3/7/8/13/14/16 are genuinely beyond the spec (undocumented, RE‑only).
- v19 quirk (per spec §2.1.5.7): ATT_MTU **≥ 247** required, and the **accessory** must send the
  `ATT_EXCHANGE_MTU_REQ` after connecting (the bike won't initiate). The accessory **must not**
  cache value handles or the CCCD — it has to re‑discover the service/characteristic on every
  connection. Pairing is advertised via a **Service Solicitation** for the `eb20` UUID.

### Documented fields (spec V1.0)

| # | field | scaling | conf |
|---|-------|---------|:---:|
| 1 | Speed | ÷ 100 = km/h — matches `98-2D` | ✓ᵃ |
| 2 | Cadence | rpm (direct) | ✓ᵃ |
| 5 | Rider power | W (direct) — byte‑exact match with `98-5B` | ✓ᵃ |
| 9 | Ambient brightness | ÷ 1000 = lux | ✓ᵃ |
| 10 | Battery SoC | % — matches `80-88` (78→66 over a ride) | ✓ᵃ |
| 11 | Time | epoch seconds | ✓ᵃ |
| 12 | Odometer | metres — matches `98-18` | ✓ᵃ |
| 17 | Bike light | 0 = invalid · 1 = off · 2 = on | ✓ᵃ |
| 21 | System locked | bool | ✓ᵃ |
| 22 | Charger connected | bool | ✓ᵃ |
| 23 | Light reserve mode | bool | ✓ᵃ |
| 24 | Diagnosis active | bool | ✓ᵃ |
| 25 | Standstill (bike not driving) | bool | ✓ᵃ |

### Undocumented fields (observed on smart system, not in spec V1.0)

| # | best guess | evidence | conf |
|---|-----------|----------|:---:|
| 8 | Assist level | 0–4, tracks `98-09` exactly | ✓ᵃ |
| 7 | **Rider torque** | ÷ 10 = Nm; = power·60/(2π·cadence). Diagnostic `98‑14` is the same signal at ÷20 (r = 1.000) | ✓ᵃ |
| 13 | **Trip average speed** | ÷ 100 = km/h; = odometer delta ÷ field 16 ride time, byte‑exact (2302 = 876 m / 137 s) | ✓ᵃ |
| 14 | **Trip max speed** | ÷ 100 = km/h; running max of field 1, exact in 83/83 samples | ✓ᵃ |
| 16 | **Trip ride time** | seconds; 1.000/s while moving, frozen while standstill (field 25) is set | ✓ᵃ |
| 15 | Odometer snapshot at ride start | updates when standstill clears | ? |
| 3 | **Per‑mode range estimates** (km) | = diag `98-57`, one byte per configured mode | ✓ᵃ |
| 18 | Shift recommendation? | pulses 1→2 or 1→3 for a few seconds while riding (derailleur bike) | ? |
| 19, 20, 27 | unknown / constants | — | — |

## Reference power meter (`char` = `2A63`)

Standard BLE Cycling Power Measurement:

```
flags (uint16 LE) · instantaneous power (sint16 LE, watts) · optional fields…
```

Power is bytes 2–3, little‑endian signed. Optional fields (pedal balance, torque, wheel/crank
revs → cadence) follow per the flag bits. Logged into the **same file** as the Bosch frames, so
Bosch rider power (`0x98‑5B`) and the meter can be aligned by timestamp for calibration.

> **First calibration result** (July 25 ride, 331 aligned samples): Bosch rider power
> (`98-5B` / LDI field 5) averages **≈ 10 % below** the reference meter (186 W vs 205 W).
> Timestamp jitter limits per‑sample correlation — treat as a preliminary offset, not a curve.

## Cross‑check against public projects (July 2026)

Compared our map with the open reverse‑engineering community. **No contradictions to our verified
rows**; several independent confirmations and a few new leads.

- **Nilogax/SmartBridge** (Android + XIAO nRF52840 → Garmin). Decodes the same `0011` channel and
  agrees byte‑for‑byte: `98‑5B` rider W, `98‑5D` motor W, `98‑5A` cadence ÷2, `80‑88` SoC,
  `98‑09` assist 0–4, `98‑18` odometer. It reads speed off **`98‑08`** (we use the always‑present
  `98‑2D`); both are ÷100 and agree, consistent with our "two speed fields." Notable: it has **no
  torque field** and gets **motor power only from `0011`** — because the official LDI has no motor
  field — then encodes the rider/motor split as ANT+/BLE **left/right power balance** for the head
  unit. It also force‑zeros speed after 7.5 s of a frozen odometer (the "ghost speed at standstill"
  quirk that breaks Garmin auto‑pause).
- **bestie‑org/BEStie** (phone‑free nRF52840, FTMS). Uses the **actual Bosch `.proto`** (nanopb) and
  bundles the official spec PDF — that's the authoritative source that confirmed our 13 documented
  LDI fields exactly, and confirmed there are no others in V1.0.
- **RobbyPee/Bosch‑…‑Garmin‑Android** (`BLEdata.md`). Same `0x30` framing. First to flag **`98‑15`
  as motor torque** (marked "possibly", ÷200) — we now confirm `98‑15` exists and tracks motor
  power (r = 0.93), and additionally identified **`98‑14` = rider torque ÷20** which he didn't have.
  ⚠️ **One disagreement:** he scales `98‑2D` speed **÷10** (from a single Strava match); our
  whole‑ride integration to the odometer is byte‑exact at **÷100**, so we keep ÷100 (his single
  sample was likely a low‑speed roll or coincidence).
- **Xunil99/ha‑bosch‑ebike** (ESPHome LDI bridge). Implements only the 13 official LDI fields — so
  it neither confirms nor refutes our undocumented 7/8/13/14/16; those remain **our** contribution
  beyond every public decoder.
- **"Nyon Unchained," arXiv 2404.12864.** Forensic teardown of the Nyon BUI350 head unit (not our
  BLE path, but adjacent): RNDIS at `172.16.35.101:5001`, LUKS userdata whose key sits in a
  cleartext partition, plaintext Wi‑Fi passwords, and — relevant here — on‑device SQLite logs a
  **`driverTorque`** column, i.e. the hardware records torque even where the wire protocols hide it.
  It also showed trips can be **forged** in the local DB and sync to Bosch's cloud unvalidated.

**Net:** everything on the card held up; we added `98‑14` (rider torque, r=1.000) and `98‑15`
(motor‑torque candidate), the full channel UUIDs, and the exact spec connection rules. The one
outside claim we reject is RobbyPee's ÷10 speed on `98‑2D`.

---

*Field map + method: **redundo.app**. LDI: Bosch Live Data Interface spec V1.0 (May 2026, embedded
`ebike_live_data.proto`). Cross‑checked against ha‑bosch‑ebike, Nilogax/SmartBridge, bestie‑org/BEStie,
RobbyPee/Bosch‑Smart‑System‑Ebike‑Garmin‑Android, and arXiv 2404.12864.*
