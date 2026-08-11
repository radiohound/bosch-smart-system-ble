# Kiox icon IDs (`IconIdEnum`)

The numeric icon set the head unit can draw, from decompiled Flow
(`com.bosch.ebike.bes3.messagebus.IconIdEnumType`, 95 values, `0`–`94`).

**Where these are used:** the `ICON_IDS` field of a `FeatureStreamingAlert` (the interactive
alert/banner, field 5) and the `iconId` of a `ButtonContent` (field 2). See
[`NAV-PROTOCOL.md` §3a/§3c](NAV-PROTOCOL.md) for the alert schema and the turn-by-turn use.

**Color is baked into the icon.** You don't set a color — you pick an icon, and the firmware draws it
in its fixed color (e.g. `WARNING_ICON` renders as a **yellow** triangle). Text has no color field, so
color on screen comes only from the chosen icon (and, on the cert-gated map, `LayerStyle`).

**Not every id renders on every head unit.** When the display can't draw an id it returns
`ICON_ERROR` in the alert response (see §3b) — a self-reporting support check. Confirmed on a **Kiox 300**:

| id | name | Kiox 300 |
|---|---|---|
| 1 | `WARNING_ICON` | ✅ renders (yellow triangle) |
| 89 | `LOCATOR_ICON` | ✅ renders |
| 91 | `NAVIGATION_ICON` | ❌ `ICON_ERROR` |

The rest are untested per-device; treat the response as the oracle. Machine-readable list:
[`data/icon_ids.csv`](../data/icon_ids.csv). Names are verbatim from Bosch, including their typos
(`NOTIFICAITON` at 45 and 78) — the **number** is what goes on the wire.

## The glyphs

![Kiox maneuver icon glyphs by id](images/kiox-icon-map.png)

*Maneuver **stand-ins** (Mapbox artwork), not the Kiox's actual firmware glyphs — see
[Getting the real Kiox glyphs](#getting-the-real-kiox-glyphs--open-help-wanted) below. Scalable
source: [`kiox-icon-map.svg`](images/kiox-icon-map.svg).*

**Provenance — this is a maneuver *key*, not a picture of the Kiox's glyphs.** The Kiox draws each
icon from its **own firmware icon set**, which is *not* in the APK (Flow just sends the id *number*
and the head unit draws it). The glyphs here are Flow's bundled **Mapbox Navigation SDK** drawables
(`res/drawable/mapbox_ic_*.xml`, decompiled Android-vector → SVG), mapped to `IconIdEnum` by maneuver
name (`1` `WARNING` = Flow's `ic_alert_warning`). **Confirmed on hardware (Kiox 300):** the id ↔
maneuver mapping is correct (a photographed id 2 is a merge, 5 is an arrive-right, etc.), but the
**drawings differ** — e.g. id 5 `ARRIVE_RIGHT` curves the arrow *into* the dot on the Kiox vs a
straight arrow here. Use this to look up **which maneuver an id is**, not what it looks like pixel-for-pixel.

**Colors are applied by the head unit.** These are **monochrome template icons** (drawn `#000000`,
tinted at runtime), so the sheet is rendered the way the Kiox actually shows them: **white on a dark
background**, with the **secondary road in gray** (`#a8a8a8` — the real `mapbox_turn_icon_shadow_color`,
so a merge/fork shows *your* route white and the *other* road gray) and **`WARNING` (1) in yellow**
(confirmed on hardware). The exact per-icon color for every id is a firmware choice not fully encoded
in the APK; what's faithful here is the geometry and the route-vs-secondary two-tone.

**88 of 95 shown.** `CONTINUE_*`, `NEW_NAME_*`, and `INVALID_*` have **no distinct Mapbox glyph** — Mapbox
(and the Kiox) reuse the matching *turn* arrow, so those ids are drawn here with the turn glyph as a proxy
(e.g. `continue left` = the `turn left` arrow). Still not pictured: `ICON1` (0), `UPDOWN` (40), `FLAG` (43),
and the status icons `NO_GPS` (90), `RECENT_SEARCH` (92), `MUSIC` (93) — no drawable for them in the APK.

## Getting the *real* Kiox glyphs — open (help wanted)

To be explicit: **the sheet above is not what the Kiox draws.** It's the Mapbox phone artwork used as a
maneuver stand-in. The head unit renders each id from **bitmaps baked into its own firmware**, in a
visibly different style (a photographed id 5 `ARRIVE_RIGHT` curves the arrow into the destination dot;
the Mapbox stand-in is a straight arrow with the dot on top). A firmware bitmap set is **not present
anywhere in the Flow APK** — Flow only ever sends the id *number*. So building the *true* icon list needs
one of two things, and **we'd welcome help with either**:

**A. Screenshot each icon (best — exact pixels, no decoding).** Display an id via a `FeatureStreamingAlert`
(we can already drive any id on demand) and capture the screen. The clean digital route is the head unit's
own **`DEBUG_TAKE_SCREENSHOT`** command (`8D-82` / `A2-80`), which returns rendered pixels — but it reads
`not-available` over **BLE**. The **wired diagnostic channel is less restricted than BLE** (it's how the
USB-only fields like pack voltage are read), so `DEBUG_TAKE_SCREENSHOT` may well answer there. *Help wanted:*
anyone with a **wired diagnostic connection** (e.g. a `bes3-reader`-style USB link) who can try `8D-82` and
see whether image data comes back. The low-tech fallback is simply **photographing the screen** for each id.

**B. Extract the Kiox firmware image and carve the bitmaps.** Flow's FOTA layer stores a direct **`downloadUrl`**
(plus `fileHash`, `fileSizeBytes`, `pki`) for each component's firmware in a local SQLite DB
(`UpdateSetInfoDatabase`, table `UpdateSetSoftware`), and Flow **does not decrypt** the image — it relays it
to the bike as-is, so the downloaded file is exactly what the bike flashes. Given the image, the icon/font
bank can be carved and rendered. *Caveats:* the URL only appears **when an update is actually offered** (a Kiox
on older firmware, or the next Bosch push), reading the DB needs a **rooted phone**, and the image is
**signed** (`pki`) and **may be encrypted** — though asset regions (icons/fonts) are often left in the clear.
*Help wanted:* a captured **Kiox firmware update `.bin`** (any version — the icons rarely change).

If you can help with a wired-channel `DEBUG_TAKE_SCREENSHOT` test, a Kiox on un-updated firmware, or a
firmware `.bin`, please open an issue. Once we have the real glyphs, this sheet gets replaced with the
actual head-unit artwork mapped by id.

## Status / UI icons

| id | name | glyph |
|---|---|---|
| 0 | `ICON1` | enum default / placeholder |
| 1 | `WARNING_ICON` | yellow warning triangle |
| 89 | `LOCATOR_ICON` | location pin |
| 90 | `NO_GPS_ICON` | no-GPS |
| 91 | `NAVIGATION_ICON` | navigation arrow/compass |
| 92 | `RECENT_SEARCH_ICON` | recent search |
| 93 | `MUSIC_ICON` | music note |
| 94 | `KOMOOT_ICON` | Komoot logo |

## Maneuver arrows (`DIRECTION_*`)

Mapbox-style `type × modifier`. Cell = icon id; `·` = no such variant. The straight column is the
"through" arrow; *base* is the modifier-less icon for that type.

| type | sharp&nbsp;L | left | slight&nbsp;L | straight | slight&nbsp;R | right | sharp&nbsp;R | base | u-turn |
|---|---|---|---|---|---|---|---|---|---|
| **turn**        | 69 | 44 | 37 | 68 | 55 | 38 | 12 | ·  | ·  |
| **continue**    | ·  | 9  | 31 | 70 | 74 | 66 | ·  | 73 | 75 |
| **depart**      | ·  | 88 | ·  | 7  | ·  | 41 | ·  | 50 | ·  |
| **arrive**      | ·  | 79 | ·  | 14 | ·  | 5  | ·  | 35 | ·  |
| **merge**       | ·  | 2  | 64 | 17 | 58 | 60 | ·  | ·  | ·  |
| **fork**        | ·  | 3  | 39 | 48 | 82 | 29 | ·  | 76 | ·  |
| **roundabout**  | 24 | 36 | 22 | 27 | 86 | 72 | 8  | 83 | ·  |
| **rotary**      | 18 | 11 | 30 | 51 | 46 | 19 | 65 | 15 | ·  |
| **on-ramp**     | 47 | 61 | 62 | 87 | 67 | 33 | 6  | ·  | ·  |
| **off-ramp**    | ·  | 20 | 10 | ·  | 28 | 34 | ·  | ·  | ·  |
| **end-of-road** | ·  | 25 | ·  | ·  | ·  | 71 | ·  | ·  | ·  |
| **new-name**    | 80 | 85 | 81 | 56 | 84 | 57 | 26 | ·  | ·  |
| **notification**| 32 | 53 | 49 | 54 | 21 | 45 | 78 | ·  | ·  |
| **invalid**     | ·  | 77 | 4  | 16 | 63 | 52 | ·  | 42 | 23 |

Standalone (no modifier): **u-turn** `59` · **close** `13` · **flag** `43` · **up/down** `40`.

### Common turn-by-turn subset

For basic guidance, the `turn` row plus a few extras cover most cases:

| maneuver | id | | maneuver | id |
|---|---|---|---|---|
| turn left | 44 | | u-turn | 59 |
| turn right | 38 | | arrive / flag | 35 / 43 |
| straight (turn) | 68 | | depart | 50 |
| continue | 73 | | roundabout | 83 |
| slight left / right | 37 / 55 | | fork left / right | 3 / 29 |
| sharp left / right | 69 / 12 | | keep left / right (fork slight) | 39 / 82 |
