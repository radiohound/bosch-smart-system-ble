# Talking to the bike — bonding, GATT, and MCSP framing

The rest of this repo is about **what** you can read: which addresses answer, what the
fields mean, why the rest refuses. This document is about **how to hold a conversation** —
the layers underneath the field map, in the order you hit them.

Everything here is from one bike — a Performance Line CX (BDU3740, PowerTube 750, fw 20.x,
Kiox 300 + BRC3600 LED remote) — using a native Android `BluetoothGatt` client and an iOS
CoreBluetooth client. Frame statistics are measured from **5,648 frames across 9 capture
files**, all on the diagnostic notify characteristic.

Not repeated here, because they have their own homes:

- **The request/response grammar, refusal codes, and the 4-bit sequence** →
  [BLE-ACCESS.md](BLE-ACCESS.md)
- **The MobileApp startup handshake (`0x40AA`/`0x40A9`)** →
  [STARTUP-HANDSHAKE.md](STARTUP-HANDSHAKE.md)
- **What each field means** → [DECODER-CARD.md](DECODER-CARD.md)

---

## 1. Bonding — you cannot skip it, and your app should not do it

**The bike rejects unbonded connections outright**, with GATT status **147**
(`GATT_INSUFFICIENT_AUTHENTICATION`). It drops the link rather than prompting. So there is
no unbonded path to the diagnostic service, and "connect and see" is not a strategy.

### The bond is an OS object, and Flow does not create it

On Android a bond is stored in the Bluetooth stack **per remote device, shared by every app
on the phone**. It is not owned by the app that caused it. An app connecting to an
already-bonded bike does not bond again — it inherits what is already there.

**Bosch's own Flow app contains no `createBond()` call at all.** What it does instead:

- `CompanionDeviceManager.associate()` with a `BluetoothLeDeviceFilter` scan-filtered to the
  bike's address — Android's system companion-device dialog, which grants *the app* device
  access. That is an association, **not** a bond.
- A broadcast receiver on `android.bluetooth.device.action.BOND_STATE_CHANGED`, which only
  **observes**. It never initiates.

The bond is therefore created by the **Android Bluetooth stack**, because the bike demands
an encrypted link. Flow contributes nothing cryptographic to it.

**Consequence: there is nothing special about a "Flow bond."** Any route that ends with the
OS bonding to the bike produces the same bond — pairing through Flow, or pairing from the
system Bluetooth settings with the bike in its pairing mode (which is what `bes3-reader`
instructs). Flow is convenient, not required.

> **Practical note.** Because an unbonded connect fails 147 rather than prompting, a client
> that refuses to call `createBond()` must give the user instructions that actually work on
> a phone which has never seen the bike. Test that path before shipping it.

### The bond lives on the remote, not the drive unit

**All 39 `BLE_*` datapoints in the ~895-address registry belong to one component:
`RemoteControl`** — the BRC3600 LED controller on this bike. None are on DriveUnit,
HeadUnit or Battery. That covers the radio, the pairing-mode flag, both bond databases, the
device address, BLE heart rate and BLE button events.

Two things follow:

- **It explains the USB/BLE split on the bond datapoints.** The USB diagnostic port is
  physically on the BRC3600, so USB attaches directly to the component that owns this data,
  while BLE reaches the same component through its gateway with the policy filter in front.
- **The firmware that governs bonding behaviour is the remote's, not the drive unit's.**
  `A0-6B RemoteControl.SOFTWARE_VERSION` and `A0-62 PART_NUMBER` both read over BLE *and*
  auto-push in the passive stream, so a listen-only connection captures them with zero
  requests. Reports that identify affected bikes by drive unit may be comparing on the
  wrong axis.

### The bond databases, and what you cannot do to them

The bike keeps two, both on `RemoteControl`:

| role | what it holds | enumerate | clear |
|------|---------------|-----------|-------|
| **central** — the bike connecting out to sensors (HR straps, buttons) | `A2-10 BLE_CENTRAL_DATABASE`, `A2-0F …GET_AVAILABLE_DATABASE_SLOTS` | over BLE | **`A2-11 BLE_CENTRAL_CLEAR_DATABASE` exists** |
| **peripheral** — phones bonded to the bike | `A2-12 BLE_PERIPHERAL_GET_AVAILABLE_DATABASE_SLOTS`, `A2-13 …GET_DATABASE_ENTRY`, `A2-14 …DATABASE_ENTRY_UPDATED` | **USB only** — all three `DENIED` over BLE | **no clear command exists anywhere in the registry** |

Separately, `CONFIGURED_SMARTPHONES` (`0x220C`) is an app-level list of configured phones —
readable and subscribable, **no write, no delete**.

So the sensor database can be enumerated and cleared; the phone-bond database can be
enumerated over USB and not cleared. **Bond-table capacity on this hardware is unmeasured**
— `A2-12` would answer it and is read-only, if someone with USB access wants to settle it.

> ⚠️ **`A3-33 RemoteControl.REMOVE_DEVICE`** answers a READ with `UNSUPPORTED` rather than
> `DENIED` over BLE, which on this bus means "reachable, wrong verb" rather than "policy
> wall", and it reads as a value over USB. It is **completely untested**
> (`confirmed: false`, `kind: unknown`) and we are not claiming it does what its name
> suggests. But note the symmetry: the one call that might make a bad bond recoverable is
> also the one most likely to destroy a good one. **Do not brute-force it on a bike whose
> pairing you care about.**

### The bond-burn hazard applies to the peer path, not this one

A documented community report (r/BoschSmartSystem, u/InvestigatorSenior) describes current
gen4 firmware bonding **once** to a phone and then never reconnecting, with no way to clear
the bond from the bike. It is real, and it is **specific to making the phone an LDI
peripheral** so that the *bike* connects to *it*.

The mechanism is why it does not generalise: in that arrangement the bike is the central and
must **initiate**, which requires resolving the phone's rotating address against the IRK
exchanged at bonding. The bike resolves RPAs while *scanning* but reportedly will not use the
IRK to *initiate* — so after the first bond it can never reach that phone again. Dedicated
LDI computers dodge it with a random **static** address: nothing to resolve.

**In the normal phone-as-central arrangement described in this document, the bike never
initiates**, so the broken step is not on the path. Two phones (Android and iOS, both on
rotating private addresses, both bonded before our clients ever ran) have reconnected to
this bike without incident — the Android client across four weeks of full-registry sweeps,
the iOS client since June.

---

## 2. GATT layer

| | |
|---|---|
| Advertised service | `0000fe02-0000-1000-8000-00805f9b34fb` |
| MCSP service | `00000010-eaa2-11e9-81b4-2a2ae2dbcce4` |
| **RX / notify** (bike → phone) | `00000011-eaa2-11e9-81b4-2a2ae2dbcce4` |
| **TX / write** (phone → bike) | `00000012-eaa2-11e9-81b4-2a2ae2dbcce4` |

(The documented Live Data Interface is a separate surface: service `eb20`, characteristic
`eb21`.)

**MTU: 247 is enough, and you must ask for it.** We request 247 and the full read path
works; we have never needed 512, and nothing suggests it is a protocol requirement. The bike
sizes frames to whatever was negotiated — see §3. Note that **the bike will not initiate the
MTU exchange**: Bosch's own LDI spec (§2.1.5.7, v19) requires ATT_MTU ≥ 247 and puts the
`ATT_EXCHANGE_MTU_REQ` on the accessory. Request it yourself, early, through the same
serialised queue as everything else.

**`0x0012` is write-without-response only.** It does not advertise `PROPERTY_WRITE` at all,
so write-with-response is not available as a safety net. Derive the write type from the
characteristic's properties rather than hardcoding it.

> ### The trap: write-without-response has no backpressure
> A naive send loop **silently discards frames**. Pushing 4 fields per 1 Hz tick, only one
> landed per tick — **66 of 88 frames gone, 22 ticks out of 22** — while every counter
> reported a healthy loop. It looks exactly like the bike ignoring you.
>
> The fix is an outbound queue drained from the platform's "ready to send" callback
> (`peripheralIsReady(toSendWriteWithoutResponse:)` on iOS; the GATT write callback on
> Android). After: **508/508 sent, zero drops over 127 ticks.**
>
> More generally: **serialise every stateful GATT operation** through one queue —
> characteristic writes, CCCD descriptor writes, `requestMtu`. And **log TX alongside RX**;
> neither root cause in the session that found this was visible from an RX-only capture.

**Connection handling, Android-specific but hard-won:**

- Always `close()` the GATT client on any disconnect or connect error, or it leaks and the
  next attempt fails with status **133**.
- After the first miss, switch from a direct connect to a standing `autoConnect = true`.
  The failure is almost always "the bike was not advertising just then", which waiting
  solves and retrying does not.
- Do not subscribe to every notify characteristic at service discovery, and **verify frames
  actually arrive** — a subscribe that "succeeds" against a silent channel is
  indistinguishable from a healthy quiet one. See [STARTUP-HANDSHAKE.md](STARTUP-HANDSHAKE.md)
  for why that silence usually happens and how to avoid it.

---

## 3. MCSP segmentation

Every frame on `0x0011` and `0x0012` carries a **2-byte segmentation header**:

```
byte0 = (channel << 5) | (endOfChannel ? 0x10 : 0) | ((len >> 8) & 0x0f)
byte1 = len & 0xff
```

The length field is 12 bits, so a frame *could* declare up to 4095 bytes. On this bike it
never does — see the sizes below.

**Channels:**

| channel | use |
|---------|-----|
| 0 | transport **command** (flow control) |
| 1 | **message bus** — all application traffic |
| 2 | LBTP pull |
| 3 | LBTP push — stored-log file transfer |

**In practice only three header bytes occur** on the diagnostic channel:

| header | count | meaning |
|--------|------:|---------|
| `0x30` | 5,621 | channel 1, end of message, len < 256 — the normal case |
| `0x10` | 19 | channel 0, end of message — MCSP flow control, **not** application data |
| `0x20` | 8 | channel 1, **not** end of message — a **continuation** |

### Three things a parser must handle

1. **Several complete frames pack into one notification.** Routine, not an edge case:
   `30 05 0096 C1807A │ 30 05 00AF C1807A │ 30 07 00D7 C0801D085B │ …`
2. **A logical message can span frames.** Continuation frames (`endOfChannel = 0`) must be
   accumulated until one arrives with the bit set. Reassembling a 242-byte continuation with
   its remainder produces a 251-byte reply whose body is a well-formed protobuf; the
   fragment alone does not parse.
3. **Walk the headers. Never split on `0x30`.** Read 2 bytes, take `len`, consume `len`
   payload bytes, repeat. Splitting on a byte value breaks on continuation frames and on any
   payload byte equal to `0x30`, and it **fails silently**. Walking the headers consumes
   **3,765 of 3,765** notifications with zero leftover bytes.

### Sizes actually observed

| | |
|---|---|
| largest single frame payload | **242 bytes** — MTU 247 minus ATT and header overhead |
| largest reassembled message | **251 bytes** |
| frames declaring a length > 255 | **none**, in 5,648 frames — the length field's high nibble was never non-zero |
| complete logical messages measured | 5,640, **none over 255 bytes** |

The connection-open config dump does **not** break this: it is ~200 *separate small
messages* arriving fast (242 messages, 172 distinct ids in the first 3 seconds, **median
message 5 bytes**), not one large one. What looks like a large burst at startup is rate and
frame packing, not message size.

> **So a 255-byte parser limit has never been exercised here — and sits four bytes above
> the largest message we have ever seen.** That is not a proven-safe ceiling, it is an
> untested one with almost no margin. This bike is a minimal BES3 configuration (one
> battery, no ABS, no ConnectModule); a fuller one may cross it. Implement reassembly and
> respect the declared 12-bit length rather than assuming a byte.

---

## 3a. Addressing — who you say you are

Every message-bus frame body opens with a 5-byte envelope:

```
<srcHi> <srcLo> <dstHi> <dstLo> <typeSeq> [payload…]
```

Three "self" addresses turn up in this ecosystem and they are easy to conflate:

| address | name | what it actually is |
|---------|------|---------------------|
| `0x4180` | `MobileAppGatewaysAddresses.E_BIKE` | the address **Flow stamps as the source** on every outgoing read, write and RPC. Confirmed from the Flow decompile (`BluetoothGateway`'s constructor default) |
| `0x4000` | `MobileAppBrokerAddress` | the address Flow answers **as**, when the bike addresses *it* as a MobileApp peer. A different role — destination-when-addressed, not source |
| `0x0E10` | — | **not a BLE concept at all.** It is the USB DiagnosticTool 3 host address, captured from a real USB session. It appears in BLE code only where a USB protocol core was reused and the address rewritten at the transport boundary |

**The bike is tolerant about which you use.** Our clients have always sent `40 80` as the
source (`30 05 40 80 <idHi> <idLo> 00`) and READ, WRITE, RPC and SUBSCRIBE all work. We also
tested the identity as a variable directly — two clients, `0x4080` and `0x4180` — and got
**identical denials on all 213 overlapping rows**. So the source address is **not a lever on
the privilege wall**; getting it "more correct" unlocks nothing. (It may still matter for
particular strict RPCs; we have no positive case either way.)

> **If you rewrite addresses at a transport boundary, mask only the low 7 bits.** The top bit
> of the echoed destination is the **success flag**, not part of the address. Forcing it to 1
> makes every `DENIED` / `NOT_READY` / `NO_ROUTE_FOUND` reply get misread as a successful
> 1-byte payload — corrupting both the ok/declined counts *and* the payload offset. This is
> not hypothetical; it was a real bug in the reference implementation, fixed once the status
> codes started arriving.

See [BLE-ACCESS.md](BLE-ACCESS.md) for what comes after the envelope — the status word, the
refusal codes, and the 4-bit shared sequence.

---

## 4. Transport control commands — optional here

The MCSP command channel defines `VERSION` (`0x01`), `ADVANCE_TRANSMIT_WINDOW` (`0x02`),
`DISABLE_FLOW_CONTROL` (`0x03`) and `MAX_SEGMENTATION_PACKET` (`0x04`).

**Our clients send none of them.** No channel-0 traffic is generated at all — they open
GATT, subscribe to `0x0011`, and write channel-1 frames directly:

```
30 05 40 80 <idHi> <idLo> 00
```

READ, WRITE, RPC and SUBSCRIBE all work, in sessions lasting the length of a 206-minute
ride. So none of the three is a precondition for normal message-bus traffic on this
firmware.

**But the bike uses channel 0 unprompted.** All 19 channel-0 frames observed carry
`02 01 00 00 02 xx` — `ADVANCE_TRANSMIT_WINDOW`. The bike runs flow control whether or not
the client negotiated it. We ignore those frames entirely and nothing breaks — **but our
request rate is low**: one request in flight, serialised, ~300 ms apart. Ignoring a
transmit-window grant may only be safe at that rate, and a bulk or parallel writer may
genuinely need to honour it. That case is untested.

The honest claim is **"demonstrably optional for a serialised, one-request-at-a-time
client"** — not "never required."

---

## 5. Session upkeep — no BLE keepalive needed

`bes3-reader` sends `RESET_INACTIVITY_SHUTDOWN_TIMER` (`0x2106`) roughly every 800 ms during
a diagnostic sweep. **That is a USB finding**: without it the bike's inactivity timer closes
the session and the controller drops off USB mid-sweep.

**On BLE it appears unnecessary.** Neither of our clients has ever sent `0x2106` — it exists
in our code only as a registry entry, with no call site — and sessions stay alive with
continuous frames for entire rides, up to **206 minutes**.

The likely reason: on BLE the bike auto-pushes telemetry continuously, so the link is never
idle in the way a USB diagnostic session is. The ~800 ms interval is in any case an
implementation choice, not a documented protocol figure.

---

## Method and scope

One bike, deeply. Frame statistics are derived mechanically from raw capture bytes rather
than from a human reading a log — 9 JSONL capture files, each line one notification with its
hex payload, plus `__PROBE_TX__` lines recording outbound frames. The TX side matters: two
root causes in §2 were invisible from RX-only captures.

Cross-bike results welcome, particularly from a fuller component set — the segmentation
sizes in §3 and the flow-control question in §4 are the two places where a different bike is
most likely to disagree.

*BLE transport findings by [redundo.app](https://redundo.app). Address registry © Remko
Weijnen, [`bes3-reader`](https://github.com/rweijnen/bosch-bes3-reader), CC BY 4.0.*
