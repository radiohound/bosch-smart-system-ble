# Assist-mode configuration — reading and writing the bike's riding modes

Everything here is **one bike, deeply verified**: a Performance Line CX (BDU3740, PowerTube 750,
FW 20.27.0), over the ordinary bonded diagnostic channel (`0x0011` notify / `0x0012` write). No
certificate, no OEM authorization, no cryptographic handshake. Addresses come from the decompiled
Flow registry; every claim below was confirmed on hardware unless tagged otherwise.

**Read [§ The refusal is silent](#the-refusal-is-silent) before writing anything.** A refused write
looks like a successful one.

---

## The fields

| addr | registry name | what it is |
|---|---|---|
| `98-4E` | `ACTIVE_ASSIST_MODES` | the four configured modes, as 10-char ConfigIds |
| `98-68` | `AVAILABLE_ASSIST_MODES_LOWER` | installed mode packages |
| `98-0F` | `AVAILABLE_ASSIST_MODES_UPPER` | **more** installed packages — see the union rule |
| `98-69` | `REQUIRED_ASSIST_MODES_LOWER` | the OEM's specified set |
| `98-3E` | `REQUIRED_ASSIST_MODES_UPPER` | empty on this bike |
| `98-09` | `ASSIST_MODE` | the **slot** currently selected, 1–4 |
| `90-8B` | `GET_ASSIST_MODE_INFORMATION` | per-mode name / colour / level |
| `90-90` | `READ_UDAM_VALUES` | one mode's current parameters |
| `90-91` | `READ_UDAM_DEFAULT_VALUES` | that mode's factory parameters |
| `90-92` | `READ_UDAM_LIMITS` | the bounds the drive unit enforces |
| `90-93` | `SET_UDAM_VALUES_PARAMETERS` | write one mode's parameters |
| `90-94` | `RESET_UDAM_VALUES` | reset one mode to `90-91` |
| `90-97` | `RESET_ALL_UDAM_VALUES` | reset every mode, no argument |

## `UdamParams` — the six fields that shape a mode

From `bes3/messagebus/UdamParams.java`, with the scalings from
`MessageBus.normalizeUdamParams`:

| # | name | scale | range on this bike (`90-92`) |
|---|---|---|---|
| 1 | `ASSIST_LEVEL` | ÷100 | 50–150 (0.50–1.50) |
| 2 | `MAXIMUM_MOTOR_TORQUE` | N·m | ≤ 85 |
| 3 | `ACCELERATION_RESPONSE` | ÷100 | 50–150 — Flow calls this "Dynamic" |
| 4 | `MAXIMUM_BIKE_SPEED` | ÷100 km/h | ≤ 3200 (32.00 km/h) |
| 5 | `MAXIMUM_MOTOR_POWER` | W | 100–600 |
| 6 | `EXTENDED_BOOST` | level | 0–10 — **a level, not a toggle** |
| 7 | `TRACTION_CONTROL` | — | in the schema, never seen on the wire |
| 8 | `DRIVE_TRAIN_TENSIONER` | — | in the schema, never seen on the wire |

**`f5` is a ceiling, not an output.** All four modes on this bike default to 600 W, which does not
mean Eco can produce 600 W — a weak mode is limited by its assist factor long before its power cap.
Do not present per-mode `f5` as "what this mode delivers".

**Factory defaults are nearly uniform.** `90-91` returned `assist 100 · torque 85 · accel 100 ·
speed 3200 · power 600` for all four modes, differing only in extended boost (1 / 1 / 6 / 8).
Consistent with UDAM being *user-adjustable* overlays on top of each mode's own firmware character.

## Read and write arguments are DIFFERENT shapes

This cost us a real mis-write. Verified against Flow's own frames:

```
read    40 80 90 90 <seq>  0a 0a "A100MAAAA0"                  <- the arg IS the ConfigId
write   40 80 90 93 <seq>  0a 0c 0a 0a "A100MAAAB0" 12 0e ...  <- ConfigId is field 1 OF the arg
```

Deriving the read shape from the write (double-wrapping it) returns **MALFORMED (status 8)** on
every config read. `90-94` takes the **bare** ConfigId, like the reads.

`90-93` has **no partial update** — it resends all six fields. A write is always "here is the whole
block", so it must be built from a matched `90-90` read of *that* mode. Filling that cache from an
unsolicited frame is how we once reset a tuned mode to factory values by accident.

## The refusal is silent

**Both a successful and a refused write reply with the `C0` success prefix.** Only the payload
distinguishes them:

```
applied    30 07 10 93 C0 80 <seq> 08 01
refused    30 05 10 93 C0 80 <seq>            <- empty payload; proto3 omits false
```

Anything checking only the status prefix will report writes that never happened.

**A write that matches the current values is also acked as success** and is indistinguishable from
an edit that never reached the frame. Compare against the live block before sending, and judge every
write by **re-reading**, never by the acknowledgement.

## The drive unit enforces its own limits

`90-92` caps `f4` at **3200** (32.00 km/h) on this bike, whose regional component is named
`20mph_US-CA-NZ`. Writing **3541** (22 mph) against that limit:

- returned `C0` with an **empty payload**, four times — the silent refusal above
- produced **no** `98-8B` push, where an accepted write produces one in the same second
- read back as **3200** on four subsequent reads

**Derestriction is foreclosed by the protocol, not by anyone's discretion.** This path cannot raise
the assist cut-off. It *can* restore the factory value, which matters because Flow cannot: Flow's
slider steps in whole mph, so its top step would be 3219 — above the declared 3200 — and once a rider
moves it down the highest Flow can return them to is 19 mph (3058). The factory value becomes
unreachable through the official app.

## `98-09` is a slot, not a mode

`ASSIST_MODE` returns **1–4**, indexing into the currently configured set. The same number means a
different mode on another bike, or on the same bike after the rider changes their four. To resolve it:

```
98-09  ->  position in 98-4E  ->  ConfigId  ->  90-8B  ->  name
```

There is no fixed `0 = OFF, 1 = ECO` mapping to hard-code.

## `90-8B` — names, colour, level

The record is nested one level inside payload field 1:

```
1 ConfigId (+ timestamp) · 2 shortName · 3 longName · 4 packed RGBA · 5 level
```

**Field 5 is non-zero only for configured modes** (0 for merely offered) — a free "is this one
active" signal. Colour is packed RGBA; eMTB+ came back `0x9643EDFF`.

**Prefer the bike's own name over any id table.** We built a table from prose and got it wrong twice:
`A100M00010` is TURBO on this bike, not an Eco variant, and `A100M00020` is SPORT.

**Use the LONG name.** This bike carries two modes both short-named `eMTB`, separated only by field 3:
`eMTB` (`A100EAAAB0`) and `eMTB-shortcrank` (`A100MAAAB0`). Flow hides the distinction entirely —
`AssistModesUtilsKt.getDisplayName` returns the short name whenever the long one is
`eMTB-shortcrank`, and `filterOneEMTBFromSelectableAssistModes` shows only one variant.

This bike's map, read from the bike itself:

`A100M00010` TURBO · `A100M00020` SPORT · `A100M00030` TOUR · `A100M00040` ECO ·
`A100M0AUTO` AUTO · `A100MAAAA0` TOUR+ · `A100MAAAB0` eMTB-shortcrank · `A100MSPIC7` eMTB+ ·
`A100ECOP37` ECO+ · `A100EAAAB0` eMTB

## Union `98-68` and `98-0F`, or you will refuse a mode the bike is running

`98-68` does **not** hold every installed mode. `98-0F` carries more, and on this bike
`A100MAAAB0` — a mode that was **configured and running** — appears only in `98-0F`. Treat the
assignable set as `98-68 ∪ 98-0F ∪ 98-4E`.

Both fields **repeat field 1 per entry**. A protobuf scanner keyed by field number keeps only the
last and collapses four modes into one. They are also nested at different depths: `98-4E` two levels,
`98-68` three (its entries carry a version).

## Riding modes are versioned firmware packages

Entries in `98-68` / `98-0F` / `98-69` are `{1 id, 2 version}` — `FotaApplicationIdentifier` in
Flow's terms. `98-4E` entries carry the id **only**, no version.

So a riding mode is not a preset; it is a separately-installable, versioned software application on
the drive unit. That explains the part-number-shaped ids, the LOWER/UPPER split, and why
`eMTB-shortcrank` exists as its own id rather than as a parameter.

*How those packages are distributed is deliberately out of scope for this repository.*

## `98-69` — the OEM's set

`REQUIRED_ASSIST_MODES_LOWER` returned exactly four ids on this bike, and `98-3E` (UPPER) returned
`C0` with an empty payload:

```
ECO · TOUR+ · TURBO · eMTB-shortcrank
```

That is the manufacturer's specified selection, and it differs from what the bike was running —
the configured four had **eMTB+ where the required set has TURBO**. Note that the short-crank eMTB
variant is a deliberate OEM choice here, not a default.

Flow reads these two addresses only in its firmware-check path, where the required set is unioned
with the installed set. It is **not** a "these modes cannot be removed" constraint — no such field
exists.

## `90-94` applies `90-91` exactly

Called once per mode on a bike with four configured modes, each returned `C0 … 08 01` and each was
confirmed by re-reading. **Every post-reset value matched the `90-91` defaults read beforehand,
field for field.** The pairing is confirmed, not assumed.

`90-97` (`RESET_ALL_UDAM_VALUES`) takes no argument and resets every mode at once.

## Clean negative: `98-74` field 3 is not per-mode capability

`MaximumAvailableMotorPower` is `{1 PRODUCT_LINE, 2 AVAILABLE_ASSIST_MODES, 3 SELECTED_ASSIST_MODE}`.
Fields 1 and 2 sit at the motor's nameplate (600 on this bike — field 1 is the right source for a
ceiling). Field 3 **moves with the selected mode**, which makes it look like it might state what that
mode can deliver.

It does not. Resolved against the configuration in force at each moment, field 3 is simply the
selected mode's **`f5` power cap echoed back** — eMTB-shortcrank returned 425 W eight times while its
`f5` was 425. Eco reports **600 W**, the nameplate, while actually producing about a third of that.

**Recorded so nobody else spends a week on it:** the bike publishes no statement anywhere of what a
mode can actually deliver. Every value it exposes is a configured *limit*.

## Safety and scope

- This is one bike. Message ids are known to differ across drive-unit generations; do not assume
  these addresses hold on a BDU3741 or BDU3143.
- The assist cut-off cannot be raised through this path; the drive unit refuses it (above).
- `90-93` resends all six fields, so a careless write can silently move a rider's cut-off speed.
  Re-read the speed field from the bike rather than taking it from a caller.
