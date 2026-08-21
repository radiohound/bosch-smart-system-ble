# The MobileApp startup handshake — how the diagnostic session actually comes up

The undocumented diagnostic channel (`0x0011` notify / `0x0012` write) is famously
timing-sensitive: subscribe at the wrong moment during the bike's power-on and the
channel stays dead for the entire power cycle, while the documented LDI keeps
streaming — so it looks like a working link with nothing to say. The usual
workaround is a delay: wait a second or two after connecting before subscribing,
and retry if it fails.

**That delay is treating a symptom. The real mechanism is a startup handshake, and
once you complete it you can subscribe immediately.**

## What the bike does

At power-on the drive unit runs a **staged startup** and drives the connected
MobileApp through it. Two MessageBus addresses in the MobileApp's own range carry
it:

| addr | field | direction |
|---|---|---|
| `0x40AA` | `MOBILE_APP_STATIC_FEATURE_PROPERTIES` | bike **reads** it from the app |
| `0x40A9` | `STARTUP_STAGE` | bike **writes** the current stage to the app |

The app must answer `0x40AA` declaring `stagedStartup = true` (protobuf field 3),
and acknowledge each `STARTUP_STAGE` write. The stage climbs
`UNINITIALIZED → STAGE5 (EARLY) → STAGE9 (FINAL)`; the bike is "initialised" at
**STAGE9**. (Field names and the stage enum are from Flow's own
`communicationstack/DefaultBikeInitialisationIndicator` / `StartupStageIndicator`.)

## Confirmed on hardware (2026-08-21, BDU3740 / Kiox 300)

Subscribing to `0x0011` at **~0.1 s after connect** — no boot-window delay — while
answering the handshake, the bike wrote:

```
STARTUP_STAGE = 5     (STAGE5 / EARLY)
STARTUP_STAGE = 7
STARTUP_STAGE = 9     (STAGE9 / FINAL — initialised)
```

and the diagnostic channel **stayed alive** — continuous frames for the whole
session, ~100+/s during boot settling to steady telemetry, no gap. An early
subscribe that would normally kill the channel survived, because the app
participated in the startup instead of subscribing mutely.

**So the boot-window kill was never about timing. It is about subscribing without
completing the MobileApp startup handshake.** A client that answers `0x40AA` and
acks `STARTUP_STAGE` can subscribe as early as it likes.

## Practical guidance

- **Subscribe early and participate.** Answer `0x40AA` with `stagedStartup = true`;
  ack the bike's `STARTUP_STAGE` writes.
- **Judge readiness by frames arriving, not by seeing STAGE9.** If you connect to an
  already-running bike you missed the boot and will never see the stage progression
  — but there is no boot window to survive either, so a plain subscribe just works.
- **Keep a re-subscribe fallback** for the rare silent channel, but it is a safety
  net, not the primary mechanism. The per-bike "learned delay" ladder is obsolete.

## Credit

The `0x40AA` / `0x40A9` startup exchange was named by **@Truuplei** in
[rweijnen/bosch-bes3-reader#3](https://github.com/rweijnen/bosch-bes3-reader/pull/3),
which adds a MobileApp startup-response lifecycle to that reader. That PR is what
prompted us to look — we had brushed against these addresses while building
heart-rate push (the app already answered `0x40AA`) but had never connected them to
the boot-window problem. Cross-checked here against the Flow decompile and confirmed
on hardware.
