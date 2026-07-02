# EV1527 / B99 433 MHz Ceiling-Fan RF Bridge for Home Assistant

Control cheap 433 MHz OOK ceiling-fan + light remotes (the **XH-XXX-24V-B99**
family and other EV1527/PT2262-class units) from Home Assistant, using an
**ESP32-S3 + CC1101** running ESPHome with the RadioLib external component.

This repo is a **transmit-only bridge**, shipped in **two variants** (see
[Two variants](#two-variants)): a stateless one that exposes one Home Assistant
button per remote command, and a stateful one that exposes native fan + light
entities with on-device assumed state. Neither variant yet receives/decodes the
physical remotes (see [Roadmap](#roadmap--known-gaps)).

It is the product of a long, dead-end-heavy reverse-engineering effort. The
[What we learned](#what-we-learned-the-hard-won-part) section documents both the
solution and the four separate things that had to be fixed to get there, plus a
detailed account of **what is proven vs. what is inferred**, so the next person
doesn't repeat the loops.

---

## Two variants

| Variant | File | What HA sees | State |
|---------|------|--------------|-------|
| **Stateless** (reference) | `ceiling-fan-radio.yaml` | 15 buttons per fan, 1:1 with RF commands | None on device — model state in HA if you want it |
| **Stateful** (flagship) | `ceiling-fan-entity.yaml` | A native **fan entity** (on/off, 6 speeds, direction, "Breeze" preset) + a **light entity** + timer/All-Off buttons per fan | Assumed state on the device, restored from flash across reboots |

Both share the same hardware, wiring, protocol values, and bench-proven transmit
engine. **Flash one variant per board** — switching variants creates a new device
in Home Assistant (delete the stale one).

Pick the **stateless** variant if you want raw primitives to script against, or
the smallest possible config to adapt to a different fan. Pick the **stateful**
variant if you want the fan to look and act like a fan in HA (voice assistants,
scenes, HomeKit bridge, the standard fan card) and accept optimistic state — see
[The stateful variant](#the-stateful-variant) for exactly what it tracks and how
it can drift.

---

## TL;DR for adopters

1. Your fan must be an EV1527/PT2262-class **OOK** remote (most cheap 433 MHz
   fan/light remotes are). Confirm via [Is my fan compatible?](#is-my-fan-compatible).
2. You must extract **your own** remote's 20-bit ID and 4-bit checksum key `K`
   (they are unique per remote). See [Capturing & decoding](#capturing--decoding-your-remote).
   This is the hard 90% of adoption — the YAML is the easy 10%.
3. Wire the CC1101 per [Wiring](#wiring), install the
   [RadioLib component](#dependencies), pick a [variant](#two-variants), paste
   your ID/K/frequency into its `substitutions:` block, and flash.

> **Note on the name.** "EV1527" here is an **inference from the
> signaling**, not a part number we read off the chip. We never decapped the
> encoder. We identified the family from its behavior — the sync-header
> requirement, the 3:1 / 1:3 OOK bit geometry, and a fixed 20-bit ID + checksum —
> which matches the EV1527/PT2262 learning-code family and `rc-switch` protocol 1.
> If you have a fan that behaves the same but uses a different chip, this will
> still likely work; if your timings differ, you'll need to re-measure them.

---

## Hardware

| Part | Used here | Notes |
|------|-----------|-------|
| MCU | ESP32-S3-DevKitM-1 | Any ESP32 with a free SPI bus + GPIOs should work; pin numbers below are S3. |
| Radio | CC1101 (E07-M1101D) | 433 MHz module. **Power from 3V3, never 5V.** |
| Fan fixture | [Ohniyou Cage 21-inch Flush-Mount Bladeless](https://ohniyou.com/products/ohniyou-cage-21-inch-flush-mount-bladeless-ceiling-fan-006-%E5%A4%8D%E5%88%B6-006%E5%A4%9A%E5%B1%9E%E6%80%A7-2-%E5%A4%8D%E5%88%B6) | The specific ceiling fan validated against. |
| Fan RF receiver + remote | `XH-XXX-24V-B99` "DC motor Control Driver" + handheld remote | The 433 MHz OOK receiver built into the fan and its stock 15-button remote — what this bridge emulates. The middle characters of the model were illegible on our unit; the `-24V-B99` family suffix is the identifier that matters. |

The **B99 receiver** is a DC-motor (ECM) fan controller wired inline between AC
mains and the fan: a mains input, a dimmable LED **LAMP** output, a 3-phase
**UVW** motor output, and a **433 MHz** OOK receiver on an **ANT** antenna
terminal. Its stock **handheld remote** carries exactly the 15 buttons of the
[command map](#command-map-5-bit-identical-across-b99-units): All Off, Light
On/Off, F/R direction, the "natural wind" variable mode, a 1–6 speed dial with a
Fan/Off center, and 1H/2H/4H timers — the full command set this bridge
reproduces.

<p align="center">
  <img src="images/ceiling-fan.jpg" alt="Ohniyou Cage 21-inch bladeless flush-mount ceiling fan" height="240">
  <img src="images/b99-remote-and-controller.webp" alt="B99 DC-motor control driver and its 15-button handheld remote" height="240">
</p>
<p align="center"><sub>Left: the fan fixture. Right: the B99 "DC motor Control Driver" receiver (note the LAMP / UVW / ANT terminals) and its stock 15-button remote.</sub></p>

### Wiring

Single-pin transmit topology (this is **required** — see
[Fix #3](#fix-3-single-pin-wiring-required-for-transmit)):

```
CC1101            ESP32-S3
------            --------
GDO0   ───────►   GPIO5     (transmit data; open-drain + pullup in YAML)
CSN    ───────►   GPIO10
SCK    ───────►   GPIO12
MOSI   ───────►   GPIO11
MISO   ───────►   GPIO13
VCC    ───────►   3V3       (NOT 5V)
GND    ───────►   GND
GDO2              (unused in this transmit-only build)
```

If you also want to *receive* the remotes for your own decoding work, GDO2→GPIO6
is a reasonable dedicated RX pin (see [Capturing](#capturing--decoding-your-remote)),
but the shipped bridge does not use it.

---

## Dependencies

- **ESPHome** (validated on 2026.6.3).
- **RadioLib external component:**
  [`github://juanboro/esphome-radiolib-cc1101`](https://github.com/juanboro/esphome-radiolib-cc1101).
  This is **not optional** — see [Fix #2](#fix-2-stock-esphome-cc1101-transmit-is-broken).
- **`esp-idf` framework** (not Arduino) — RadioLib needs it here.
- A `secrets.yaml` providing `wifi_ssid`, `wifi_password`, and
  `esp_433_radio__encryption_key`.

---

## Files

| File | Role |
|------|------|
| `ceiling-fan-radio.yaml` | The **stateless** variant: two fans (Office, Guest) as repeated button blocks sharing one transmit engine. The minimal reference — also the protocol-provenance record. |
| `ceiling-fan-entity.yaml` | The **stateful** variant: native fan + light entities per fan, on-device assumed state, same transmit engine. See [The stateful variant](#the-stateful-variant). |

> Both configs are intentionally **single readable flat files** with some
> repetition between fans, rather than clever multi-file packages. See
> [Scaling up](#scaling-up--modularity) for the modular approach if you outgrow
> that.

---

## Setup

1. **Install the RadioLib component** — it's referenced via `external_components:`
   in the config, so ESPHome fetches it on first compile. Nothing manual needed
   beyond having network access at build time.

2. **Capture your remote's ID and K** — see
   [Capturing & decoding](#capturing--decoding-your-remote). You cannot skip this;
   the values in the repo (`0x003B9`/`K=0xB`, `0x18999`/`K=0xA`) are *our* remotes.

3. **Edit `substitutions:`** in your chosen [variant](#two-variants):
   ```yaml
   substitutions:
     office_id_dec: "953"        # YOUR 20-bit ID, in DECIMAL (0x003B9 = 953)
     office_k:      "11"         # YOUR checksum key, in DECIMAL (0xB = 11)
     office_freq_mhz: "433.92"   # carrier frequency
   ```
   Note the values are **decimal**, with the hex shown in a comment. Convert your
   captured hex ID/K to decimal.

4. **Flash** via the ESPHome dashboard (Install → OTA or USB).

5. **Stateless variant**: the device exposes one HA button per command (e.g.
   `button.office_light`, `button.office_speed_3`, `button.office_all_off`) —
   drive them directly or build HA-side logic on top (see
   [Architecture](#architecture-why-stateless)). **Stateful variant**: the device
   exposes `fan.office_fan`, `light.office_light`, and timer/All-Off buttons (see
   [The stateful variant](#the-stateful-variant)).

---

## Architecture: why stateless

*(This section explains the **stateless variant**'s philosophy. The
[stateful variant](#the-stateful-variant) is its deliberate counterpoint: it
accepts the desync risks described here in exchange for native fan/light UX,
and mitigates them explicitly.)*

In the stateless variant the ESP is a **pure transmitter**. Each button emits
exactly one RF frame and the device tracks no fan/light state. This is
deliberate:

- **A transmitter cannot observe the fan.** Any state the firmware claims is a
  guess that desyncs the moment someone uses the physical wall remote or the fan
  loses power. Pushing state into Home Assistant (where it can be modeled with
  helpers and corrected) keeps the firmware simple and reliable.
- **The fan/light protocol is mostly absolute, except the light.** Speed 1–6,
  Fan Off, Forward, Reverse are *absolute* commands. The **Light command is a pure
  toggle** — there is no discrete "light on" / "light off" in this protocol. So
  even with perfect state tracking, the light can only ever be tracked
  optimistically. This is a protocol limit, not a design shortcut.

If you're running the stateless variant and want a single "Off / Fwd 1–6 /
Rev 1–6" control with (optimistic) state, build it **in Home Assistant** with an
`input_select` + an automation that calls these buttons — or just run the
[stateful variant](#the-stateful-variant), which does that modeling on-device.

### The rolling counter (the one piece of retained state)

The 32-bit frame carries a 3-bit counter that the **physical remotes increment by
one per button press** and hold constant across the ~10 repeats of a single press.
This repo keeps a **per-fan counter** (`ctr_office`, `ctr_guest`) and advances it
on each transmit, so a controller that validates the counter will accept the
frames. This is the only device-side state, and it's part of the *protocol*, not
the UX.

> **What we don't definitively know:** whether the fan actually *validates* the
> counter or ignores it. We kept per-fan counters for fidelity to the real remotes
> rather than because we proved the fan rejects a static counter. If you want
> maximum simplicity you could use a single shared counter; we chose fidelity.

---

## The stateful variant

`ceiling-fan-entity.yaml` turns each fan into a native ESPHome **fan entity**
(on/off, 6-step speed slider, forward/reverse, a "Breeze" preset for the
Variable command) plus a **light entity**, with **assumed state** kept in
flash-restored globals on the device. Native entities mean the standard HA fan
card, scenes, voice assistants, and the HomeKit bridge all work without any
HA-side config.

**What HA sees per fan:** `fan.<name>_fan`, `light.<name>_light`, and buttons
for Timer 1h/2h/4h, All Off, and a diagnostic "Light State Invert".

**How it transmits.** All fan logic lives in one *reconciler* per fan: on every
entity state change it diffs the commanded state against the assumed physical
state and transmits only the differences (the protocol is mostly absolute, so
each difference is one frame). Transmissions are serialized through a queued
script (~450 ms apart) so slider drags and multi-frame settles can't overlap on
the air.

**Honesty about "assumed" state.** A transmitter still can't observe the fan;
this variant *chooses* optimistic tracking and mitigates the drift paths:

| Drift path | Behavior / remedy |
|---|---|
| Light (protocol-level **toggle**, no discrete on/off) | Entity transmits the toggle only when commanded state ≠ assumed state. If it drifts (e.g. someone used the wall remote), press **Light State Invert** — flips the assumption without transmitting. |
| Timer buttons | Fire-and-forget; when the fan's own timer fires, the fan entity stays "on" until your next command. Deliberate — timer-cancellation semantics are unverified. |
| Fan power cycle | Desyncs everything until the next command. Unavoidable without RX. |
| Anything else | **All Off** transmits the atomic all-off frame and force-syncs fan + light entities to off — it is the manual resync-to-known-state affordance. |

**Boot behavior: nothing is ever transmitted at boot.** A suppress guard boots
armed-off, the saved assumed state is pushed into the entities silently, and
only then is transmit enabled. OTA updates and reboots never disturb a running
fan.

**Direction while off** is displayed immediately but transmitted on the next
turn-on (as speed-then-direction, ~450 ms apart) — whether a direction frame
starts a stopped fan is unverified, so we don't risk it.

**Bench-verify before trusting** (open behavioral questions, checklist order):
does Speed N start a stopped fan; boot silence across reboots; state restore
across a hard power cycle; the All Off trace showing exactly one transmitted
frame.

---

## What we learned (the hard-won part)

Getting this fan to actuate took **four separate fixes**, discovered in this order.
Any one missing = nothing works. If you're debugging a non-working build, check
them in this order.

### Fix #1: The CC1101 antenna

Our module shipped with an **unsoldered antenna connection**. This silently
crippled both RX and TX (~30 dB down) and **invalidated every transmit test we did
before noticing it.** If your transmissions don't actuate anything, verify the
antenna is physically connected *before* touching anything in software. RF
debugging order is always: **antenna first**, then modulation, then framing, then
timing.

<p align="center">
  <img src="images/cc1101-unsoldered-antenna.jpg" alt="CC1101 E07-M1101D module with the SMA antenna connector pads unsoldered" width="300">
</p>
<p align="center"><sub>Our CC1101 (E07-M1101D). The SMA antenna connector's pads shipped unsoldered — about 30 dB down until repaired.</sub></p>

### Fix #2: Stock ESPHome CC1101 transmit is broken

ESPHome's built-in `cc1101` + `remote_transmitter` async-OOK transmit path is
silently broken
([ESPHome issue #16876](https://github.com/esphome/esphome/issues/16876)): the
pin-mode handling severs the RMT peripheral from GDO0, so the chip enters TX but
the keying never reaches the pin. **The fix is to drive the CC1101 via the
RadioLib external component instead.** This is why the dependency is mandatory.

> **Proven.** With RadioLib, transmit works; with the stock component, it didn't.

### Fix #3: Single-pin wiring (required for transmit)

With the RadioLib component on this hardware, transmit only works with
**single-pin wiring**: GDO0 = GPIO5 carries the data, configured **open-drain +
pullup**. A dual-pin attempt (separate GDO2 = GPIO6) would **not actuate** the fan.

> **Proven for transmit on our hardware.** We did *not* exhaustively prove a
> dedicated RX-on-GDO2 path fails for *receive* — only that splitting TX off GPIO5
> broke transmit. If you only need TX (this repo), use single-pin and move on.

### Fix #4 — THE DECIDER: the EV1527/PT2262 sync header

This is the one that cost the most time. After frequency, bitrate, and exact bit
timing were all verified and ruled out, the fan *still* ignored payload-perfect
frames that a separate CC1101 listener decoded cleanly.

**The cause: our frames were missing the sync header.** EV1527/PT2262-class
receivers require a **sync element on every frame** — a short HIGH pulse followed
by a long LOW gap — which the decoder uses as the frame delimiter. Our earlier
transmissions had inter-repeat *dead air* (a gap) but **no leading sync HIGH
pulse**, so the fan never registered a frame boundary. A generic OOK listener
decodes the data bits fine (it only cares about pulse widths), which is why the
listener said "perfect" while the fan disagreed — and why the *remote's* capture
looked "noisy" to the listener (it was choking on exactly the sync/repeat
structure the fan needs).

The fix: prepend `[250 µs HIGH][long LOW gap]` before each of the 10 repeated
32-bit frames.

**Empirically measured sync-gap threshold (on our fan):**

| Sync gap | Result |
|----------|--------|
| 3750 µs  | **FAILS** |
| ≥ 4000 µs | **WORKS** |

We use **7750 µs** in production (the EV1527 31× short-bit value), which clears the
threshold with comfortable margin. Don't run near the 4000 µs edge — leave margin
for jitter / unit-to-unit variation.

> **Proven on our fan.** The 4000 µs threshold is *our unit's* measurement. A
> PT2262-based unit's gap is set by an external resistor and could differ. If your
> fan ignores everything but a generic listener decodes your frames, suspect the
> sync gap and sweep it (see the sweep approach in the git history).

---

## The protocol (validated)

32-bit OOK frame at **433.92 MHz**:

```
[ 20-bit ID ][ 5-bit command ][ 3-bit counter ][ 4-bit checksum ]
  bits 31..12    bits 11..7        bits 6..4         bits 3..0
```

**Bit encoding (OOK, "mark" = carrier on, "space" = carrier off):**

| Bit | Mark (HIGH) | Space (LOW) | Ratio |
|-----|-------------|-------------|-------|
| `1` | 759 µs | 243 µs | ~3:1 (long-high / short-low) |
| `0` | 251 µs | 749 µs | ~1:3 (short-high / long-low) |

**Sync header (prepended to every frame):** 250 µs HIGH + ≥4000 µs LOW (we use
7750 µs). **10 repeats** per press, all carrying the same counter value.

**Counter:** increments mod 8 per press; constant across the 10 repeats.

**Checksum** (low nibble, bits 3..0):
```
checksum = nibble(bits 11..8) XOR nibble(bits 7..4) XOR K
```
where `K` is a per-remote 4-bit key. (For bits 11..8 / 7..4, take the command +
counter field as two nibbles.)

### Command map (5-bit, identical across B99 units)

| Command | Hex | Decimal | | Command | Hex | Decimal |
|---------|-----|---------|-|---------|-----|---------|
| All Off | `0x06` | 6 | | Speed 1 | `0x10` | 16 |
| Fan Off | `0x16` | 22 | | Speed 2 | `0x12` | 18 |
| Light (toggle) | `0x08` | 8 | | Speed 3 | `0x1C` | 28 |
| Forward | `0x04` | 4 | | Speed 4 | `0x0A` | 10 |
| Reverse | `0x11` | 17 | | Speed 5 | `0x0F` | 15 |
| Variable | `0x15` | 21 | | Speed 6 | `0x0C` | 12 |
| Timer 1h | `0x02` | 2 | | Timer 2h | `0x09` | 9 |
| Timer 4h | `0x19` | 25 | | | | |

**Light is the only true toggle.** All others are absolute.

### Worked checksum example (verify your decoder against this)

Our **office** remote: ID `0x003B9`, `K = 0xB`. A **Light** frame (command `0x08`)
at counter `0`:

```
frame = 0x003B940F
        └─ ID 0x003B9, cmd 0x08, counter 0, checksum 0xF
```

Check: build the command+counter field `v = (0x08 << 7) | (0 << 4) = 0x400`. The
two nibbles the checksum uses are `nibble(bits 11..8) = 0x4` and
`nibble(bits 7..4) = 0x0`. Then `0x4 XOR 0x0 XOR 0xB = 0xF`. ✓

(Worked for counter 0 because it's the cleanest to follow. At counter 4 the field
is `0x440`, nibbles `0x4` and `0x4`, so checksum `0x4 XOR 0x4 XOR 0xB = 0xB` and
the frame is `0x003B944B`. Each press increments the counter, so the checksum
changes with it.)

If your decode of a *known* button on *your* remote produces a self-consistent
checksum with a single constant `K` across several captures, you've found your `K`.

---

## Is my fan compatible?

This repo works if your remote is **OOK** (on-off keyed, i.e. amplitude — not FSK)
and **EV1527/PT2262-class**. Quick gate:

- Capture the remote (next section) and look at the bit structure. If you see a
  **short-HIGH / long-LOW = 0** and **long-HIGH / short-LOW = 1** pulse-pair
  pattern, with a **long low gap** delimiting repeats, you're in the family.
- If your capture shows frequency-shift (two tones) rather than on/off amplitude,
  it's **FSK** and this repo's OOK timing won't apply.
- Frame width, ID length, and command codes may differ from our B99 even within
  the OOK family — you'll decode your own values regardless.

If it doesn't match, stop here; this won't work without re-deriving the protocol.

---

## Capturing & decoding your remote

You need your remote's **20-bit ID** and **checksum key `K`**. Two capture methods
worked for us.

### Method A — CC1101 in receive mode (no extra hardware)

Since you're already building a CC1101 rig, you can use it to *receive*. Put the
CC1101's RX data on a dedicated pin (GDO2 → GPIO6) and run a `remote_receiver` with
raw dumping:

```yaml
remote_receiver:
  id: rf_receiver
  pin:
    number: GPIO6
    mode: { input: true, pullup: true }
  # idle MUST be shorter than the inter-frame sync gap (≈7750 µs) but longer than
  # the longest within-frame space (≈750 µs), or all repeats merge into one
  # oversized capture and the RMT buffer overflows. ~5500 µs works.
  idle: 5500us
  filter: 120us       # reject sub-bit noise glitches
  buffer_size: 10kb
  tolerance: 35%
  dump: raw
```

> **Gotcha we hit (documented so you don't):** if `idle` is set *longer* than the
> sync gap, every repeat concatenates into a single capture that overflows the
> buffer — you'll see a flood of `Buffer overflow` and **zero** decoded frames.
> That flood means the radio *is hearing the remote* — it's a frame-delimiting
> problem, not a reception failure. Lower `idle` below the sync gap.

Press a button near the device, read the raw pulse list from the logs, and decode:
each **mark** (HIGH) > ~500 µs is a `1`, else `0`; assemble 32 bits MSB-first, then
split per the [frame layout](#the-protocol-validated). The repeats are noisy at the
edges — look for the cleanest 32-mark frame.

### Method B — RTL-SDR + Universal Radio Hacker (URH)

If you own an SDR (we used a NooElec NESDR), this is more turnkey for *seeing* the
full envelope including the sync gap that a CC1101 listener tends to mangle:

1. Tune URH to ~433.92 MHz, record while pressing a button.
2. URH's auto-detection usually recovers the bit string; set the samples/symbol so
   the short pulse ≈ 250 µs.
3. Read the ID/command/counter/checksum directly off the demodulated bits.

URH is the better tool for **first** characterizing an unknown remote (it shows the
sync structure plainly); the CC1101 method is fine once you know what you're
looking for and want zero extra hardware.

### Finding K

`K` is constant for a given remote. Capture **several** presses of known buttons,
compute `nibble(11..8) XOR nibble(7..4) XOR checksum` for each — that value **is**
`K`, and it should be identical across all captures. If it isn't constant, your
bit alignment is off.

---

## Scaling up / modularity

The shipped config is a flat file with one button block per fan — readable, and for
1–3 fans the repetition is fine. If you're running many fans (or many of these
boards), you can refactor the per-fan button set into an **ESPHome package**
(`!include` with `vars:`) so the command map lives in one file instantiated per
fan, and the main file just lists fans. We prototyped this; it's genuinely cleaner
for maintenance but adds package/substitution concepts that make first-time
adoption harder, so we kept the published default flat.

Two gotchas if you go the package route:
- **Per-instance globals must be uniquely named** (e.g. `ctr_${slug}`), because
  ESPHome merges packages by component id — same id included twice collides.
- **`${substitution}` inside an inline flow-map `{ }` breaks the YAML parser.** Use
  block-style `script.execute:` in any included file. (This bit us; it's why the
  button actions are block-style throughout.)

---

## Roadmap / known gaps

- **No RX state-tracking (yet).** Both variants transmit only. Decoding the
  physical wall remotes so state reflects manual presses is feasible (we proved
  the CC1101 hears them) and both our fans are now verified on the same
  433.92 MHz carrier, so a single parked receiver could cover them. The stateful
  variant already ships the **seams**: a transmitting flag for self-echo
  gating, a suppressed state-sync path (the All Off pattern *is* the future RX
  handler pattern), a counter-based press-dedup plan, a pinned component
  commit, and a commented-out same-pin `remote_receiver` block (the component's
  own examples share GDO0 for TX and RX — no rewiring expected). Still deferred
  pending bench validation. The radio parks in RX/idle between transmits so it
  doesn't hold a band keyed.
- **The light is toggle-only.** No amount of work makes light state authoritative
  without RX feedback, and even with RX it self-heals only on the next press. Plan
  HA-side light state as optimistic.
- **Half-duplex.** The radio is deaf during the ~330 ms it transmits. Irrelevant
  for a pure transmitter; relevant if you later add RX.

---

## What's proven, inferred, and unknown

**Proven (measured/observed on our hardware):**
- RadioLib transmit works where the stock ESPHome component doesn't.
- Single-pin wiring transmits; splitting TX off GPIO5 broke transmit.
- The sync header is required; ≥4000 µs gap works, 3750 µs fails (our unit).
- The 32-bit frame layout, bit timings, and checksum formula (verified against
  many captures decoding self-consistently).
- Our two remotes' IDs and K values.

**Inferred (consistent with evidence, not directly confirmed):**
- That the encoder is specifically EV1527/PT2262 (identified by *behavior*, not by
  reading the chip).
- That the 4000 µs threshold and 433.92 MHz generalize to other units of this
  family (likely, but re-measure for yours).

**Unknown / not tested:**
- Whether the fan validates the rolling counter or ignores it (we kept per-fan
  counters for fidelity regardless).
- Whether a dedicated RX-on-GDO2 path is fully reliable over long runs (out of
  scope for this transmit-only build).

---

## Credits

- RadioLib CC1101 ESPHome component:
  [juanboro/esphome-radiolib-cc1101](https://github.com/juanboro/esphome-radiolib-cc1101).
- The "missing header" insight was corroborated by a Home Assistant community
  thread describing the identical receive-works/transmit-fails symptom on
  ESP32+CC1101, diagnosed as a missing sync header.
