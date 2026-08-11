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

*(scalable source: [`kiox-icon-map.svg`](images/kiox-icon-map.svg))*

**Provenance / caveat.** The Kiox draws each icon from its own **firmware** — those exact bitmaps are
*not* in Flow, so this sheet can't be a screenshot of them. What it *is*: Flow's bundled **Mapbox
Navigation SDK** maneuver drawables (the arrows Flow renders on the phone), decompiled from the APK
(`res/drawable/mapbox_ic_*.xml`, Android vector → SVG) and mapped to `IconIdEnum` by maneuver name.
So each glyph is a **faithful stand-in for the maneuver** at that id — same turn, possibly a slightly
different style than the Kiox's own drawing. `1` (`WARNING`) is Flow's own `ic_alert_warning`.

**65 of 95 shown.** Not pictured: `CONTINUE_*`, `NEW_NAME_*`, and `INVALID_*` — Mapbox ships **no
distinct glyph** for these; it reuses the matching *turn* arrow (so e.g. `continue left` looks like
`turn left` = 44). Also absent from the Mapbox set: the status icons `LOCATOR` (89), `NO_GPS` (90),
`RECENT_SEARCH` (92), `MUSIC` (93), `KOMOOT` (94), plus `ICON1` (0), `UPDOWN` (40), `FLAG` (43).

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
