# The bike's issue log: reading it, and three ways to read it wrong

Every Bosch Smart System component keeps a list of **active issues** — a numeric code,
when it last occurred, and how many times. It is the closest thing the bike has to a
service history, and it is readable by the owner over USB-C without a dealer tool.

This doc covers how to ask for it, the record's wire format, and three traps that all
produce **plausible wrong answers rather than errors** — which is what makes them worth
writing down.

Everything here is verified on **one bike** — a Performance Line CX (BDU3740, PowerTube
750, one Kiox 300, fw 20.x).

## Asking for it

The query is an RPC, `EXECUTE_INFORMATION_MANAGER_COMMAND_*`, present on every
component. It exists at five access tiers, and the tiers split exactly along the
two-role boundary described in [TWO-TRANSPORT-REACHABILITY.md](TWO-TRANSPORT-REACHABILITY.md):

| tier | over BLE | over USB |
|---|---|---|
| `_RIDER` | `value` | **`DENIED`** |
| `_IBD`, `_OEM`, `_SP`, `_BOSCH` | — | **`value`** |

The dealer tiers answer the diagnostic (USB) role and refuse the MobileApp (BLE) role;
`_RIDER` does the reverse. So a phone cannot read the log this way, and the USB port
can — another instance of the same role split, not a privilege ladder.

Two commands are enough, and both are read-only:

```
GET_ISSUE_COUNT      = 4   -> entryCount
READ_ISSUE_BY_NUMBER = 5   -> one record, by entry number
```

Call `GET_ISSUE_COUNT`, then `READ_ISSUE_BY_NUMBER` across the returned range.
`returnValue = 1` (`DOES_NOT_EXIST`) is the genuine end-of-list marker — not a timeout,
not a refusal.

## The record

Field numbers confirmed against DiagnosticTool 3's own generated protobuf classes:

```
ExecuteInformationManagerCommandReturn
  1 (len)     IssueDataFrame
                2 (len)     Timestamp
                              1 (varint)  value            <- sint64, ZIGZAG
                3 (varint)  activationCount                <- occurrence count
  2 (len)     ExecuteInformationManagerCommandParameters   (echo of the request)
                2 (len)     IssueId
                              1 (varint)  value            <- the issue code
  3 (varint)  returnValue      0 SUCCESS, 1 DOES_NOT_EXIST
  4 (varint)  entryCount
```

Note the issue code comes back in the **echo of your own request** (field 2), not in the
data frame.

## Trap 1 — the list is 0-indexed

`entryCount` is *N*; valid entry numbers are **0 … N−1**.

A `1 … N` loop looks correct and behaves plausibly: it returns N−1 real records and then
gets a clean `DOES_NOT_EXIST` at *N*, which reads like a tidy end-of-list. What it
actually did was **silently drop entry 0 — the oldest record on the bike.** On the test
bike that entry was the factory/assembly event, the single most interesting row in the
log.

## Trap 2 — zigzag doubles the value, so every date overflows int32

`Timestamp.value` is a protobuf **`sint64`**, so it is zigzag-encoded. Un-zigzag it
(`n % 2 === 0 ? n/2 : -(n+1)/2`) and the result is **plain Unix epoch seconds**.

The trap is that zigzag *doubles* the number. The raw varint is therefore ~2× the epoch,
which for any modern date **always** exceeds 2³¹:

| entry | epoch (s) | raw varint (2n) | > 2³¹ |
|---|---:|---:|---|
| 0 | 1683878570 | 3367757140 | yes |
| 1 | 1774917251 | 3549834502 | yes |
| 2 | 1785806902 | 3571613804 | yes |
| 3 | 1787326589 | 3574653178 | yes |

So a varint reader built on JavaScript `<<` / `|=` — which coerce to **signed** 32-bit —
does not mangle rare edge cases. It mangles **every timestamp on this bus**. Accumulate
with `result += (b & 0x7f) * Math.pow(2, shift)` instead.

The failure has a signature: a raw `3,572,967,206` un-zigzags to `1,786,483,603`
(2026-08-11), but read through `<<` it wraps to `−722,000,090` and renders as
**1947-02-14**. **A Bosch date that comes out around 1947 is this bug, not a bad read.**

## Trap 3 — catalog codes are hex strings, not decimal

The bike returns the code as a number. Bosch's catalog keys it as a **hex string with no
prefix** — `1F0041`, and entries like `10000A` make the encoding unambiguous.

Look up `0x1F0041` as decimal `2031681` and you get a clean miss. Worse, a *wrong* hit
can look convincing: **309 of the catalog's 1070 entries share the same "this error code
is no longer used… can be ignored" text**, so a bad match lands on generic-looking prose
roughly a third of the time. Verify the key encoding before believing a lookup.

The catalog's severity field takes four values:

| severity | entries |
|---|---:|
| `INFO` | 45 |
| `WARNING` | 402 |
| `ERROR` | 597 |
| `CRITICAL_ERROR` | 26 |

## A worked example — the whole log on the test bike

```
Battery        0 issues
DriveUnit      0 issues
HeadUnit       0 issues
RemoteControl  4 issues
  #0  0x132000  count=1  2023-05-12T08:02:50Z
  #1  0x113161  count=2  2026-03-31T00:34:11Z
  #2  0x113141  count=1  2026-08-04T01:28:22Z
  #3  0x1F0041  count=6  2026-08-21T15:36:29Z
```

All four resolve at Rider tier to *"This error code is no longer used in the latest
software version and can be ignored"*, component "Control unit" — that is a real
classification here, not a failed lookup, though see Trap 3 before trusting any single
match.

**Entry #0 is the useful one.** It decodes to **2023-05-12**, and this bike's
`DriveUnit.OEM_MANUFACTURING_DATE` reads **12.05.2023**. A factory/assembly event landing
exactly on the manufacturing date is independent confirmation that both the epoch and the
zigzag decode are right — the check that turns "the numbers look like dates" into "the
decode is correct."

All four entries sit on `RemoteControl` (the BRC3600), with the drive unit, battery and
head unit each reporting zero.

## Does connecting a diagnostic tool leave a trace?

Worth knowing before you plug anything in: **on this bike, no.**

Attaching a diagnostic session raises a notification on the Kiox — a blue circle-**i**
over the word `DiagnosticTool`, the `INFO` tier, Bosch's lowest severity. It is the bike
telling the rider a service tool is connected, and it clears when the session ends.

It is **not written to the issue log.** Across roughly eight USB sessions in one day, the
counts and timestamps above never moved; a power-cycle followed by two fresh attaches
left them unchanged as well, which also rules out "logged once per power cycle." No
`DiagnosticTool` entry exists on any component.

(`0x1F0041`'s most recent occurrence happens to fall on the same day, eleven minutes
after an unrelated session. Its count of 6 is lifetime, it never moved during any
subsequent session, and it remains unexplained.)

## Getting text for a code

This repo does **not** ship Bosch's issue catalog, and neither does
[`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader) — that text is Bosch's
copyrighted content, not research output. Extract your own from software you already have
legitimate access to; `bes3-reader`'s `docs/diagnostics.md` documents the recipe (Flow's
APK carries `issues_<lang>_Rider.json`; DiagnosticTool 3 carries the richer `IBD`/`OEM`/
`SP` tiers). Without a catalog you still get codes, counts and dates — everything above
was derived that way.

## Timestamps: UTC or local?

Treated as UTC throughout, which is consistent with everything observed. It is **not
proven**: the bike also carries `RemoteControl.TIME_ZONE` (`America/Los_Angeles` here)
and `LOCAL_TIME_OFFSET`, and none of the four entries distinguishes the two. Settling it
needs a co-capture against a known wall-clock event.

---

Scope, as everywhere in this repo: **one bike, deeply verified.** A different component
set or firmware generation would change the rows, though not the wire format or the traps.

*Issue-log reachability and decode results by [redundo.app](https://redundo.app). Address
registry © Remko Weijnen, [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader),
CC BY 4.0.*
