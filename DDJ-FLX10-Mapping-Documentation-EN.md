# Pioneer DDJ-FLX10 — Updated Technical Documentation (XML/JS Audit)

## 1. Document purpose

This documentation describes the **actual state of the files provided**, based on a combined analysis of the XML, the JavaScript, and the original documentation file.  
It documents the mapping **as it is currently declared**, including the inconsistencies found between XML and JS.

## 2. Sources analyzed

- `Pioneer-DDJ-FLX10.midi.xml`
- `Pioneer-DDJ-FLX10-scripts.js`
- `DDJ-FLX10-Mapping-v1.0-Documentation.md`

## 3. Mapping identity

| Field | Observed value |
|---|---|
| XML name | `DDJ-FLX10 PROD` |
| Author | `Marc Zischka` |
| Declared compatibility | `Mixxx 2.3.6+` |
| Version in XML | `v1.0` |
| Version in JS header | `v0.2` |
| Documentation status | documentation revision based on the analyzed state |
| Revision date | 2026-03-16 |

> Important note: the internal version numbers are not consistent across the files.  
> This document therefore uses the notion of an **analyzed state** rather than a single theoretical version number.

## 4. Audit statistics

| Item | Value |
|---|---|
| XML inputs (`<control>`) | 373 |
| XML outputs (`<output>`) | 28 |
| JS bindings declared in XML | 65 |
| Declared JS functions | 38 |
| JS functions effectively called by the XML | 12 |
| Orphan XML bindings (script-binding without a matching JS function) | 3 |
| Detected MIDI collisions (`status` + `midino`) | 10 |

## 5. Current architecture

### 5.1 XML / JS split

The mapping broadly follows this logic:

- **XML** for simple native Mixxx controls: play, cue, hotcues, EQ, volume, PFL, keylock, slip, reloop, navigation, UI, effects, samplers.
- **JS** for multi-state or context-dependent behavior:
  - **SHIFT** state handling,
  - **14-bit tempo** through MSB/LSB,
  - **jog touch / scratch / bend**,
  - **reverse** with slip on hold and toggle on SHIFT,
  - **remaining / elapsed time** toggle,
  - **rate range** cycling.

### 5.2 Effective JS whitelist actually used by the XML

These are the only JS functions that are both called from the XML and resolved in the script:

1. `shiftHandler`
2. `wheelTouch`
3. `wheelTurn`
4. `rate_msb`
5. `rate_lsb`
6. `LoopHalveShift`
7. `LoopDoubleShift`
8. `reverse`
9. `TimeTypeChange`
10. `rate_reset`
11. `RangeSelector`
12. `syncKeyHandler`

### 5.3 Orphan XML bindings

The following bindings are declared as `script-binding` in the XML, but **no matching JS function exists** in the JavaScript file:

- `beatjump_forward`
- `beatjump_backward`
- `beatsync`

In their current form, these three entries are inconsistent and should be treated as **to be verified / fixed**.

### 5.4 JS functions present but not called from the XML

#### Lifecycle or internal helper functions
- `init`
- `shutdown`
- `_getDeckFromGroup`
- `_updateRate`
- `_updateTimeMode`
- `sensitivityMinimizer`
- `sensitivityMaximizer`
- `updateLEDs`

#### Legacy or alternate handlers not used by the current XML
- `shift`
- `loopIn`
- `loopOut`
- `browseNavigation`
- `syncKey`
- `cue`
- `play`
- `Play`
- `pfl`
- `loopOutButton`
- `loopInButton`
- `reloop`
- `sync`
- `playPause`
- `cueButton`
- `loopManual`
- `autoLoop`
- `reverseHandler`

## 6. Functional mapping summary

### 6.1 Common deck commands (Decks 1 to 4)

| MIDI | Logical control | Implementation | Notes |
|---|---|---|---|
| `0x00` + `0x20` (CC) | Tempo fader MSB/LSB | JS | through `rate_msb` / `rate_lsb` |
| `0x0B` | Play | Native XML | `play` |
| `0x0C` | Cue | Native XML | `cue_default` |
| `0x10` | Loop In | Native XML | `loop_in` |
| `0x11` | Loop Out | Native XML | `loop_out` |
| `0x14` | Reloop | Native XML | `reloop_toggle` |
| `0x15` | Reverse | JS | hold = slip reverse, SHIFT = reverse toggle |
| `0x1C` | Keylock | Native XML | `keylock` |
| `0x35` | Quantize | Native XML | `quantize` |
| `0x36` | Jog touch | JS | enables / disables scratch |
| `0x3E` | Time display | JS | toggles `show_seconds_elapsed` |
| `0x3F` / `0x4B` | Shift | JS | 4 deck buttons + 1 master button, global SHIFT state |
| `0x40` | Slip | Native XML | `slip_enabled` |
| `0x41` | Tempo default | Native XML | `rate_set_default` |
| `0x46` | Tempo reset | JS | also realigns the internal fader state |
| `0x54` | PFL | Native XML | `pfl` |
| `0x58` | Sync enabled | Native XML | `sync_enabled` |
| `0x60` | Rate range | JS | cycles through several ranges |
| `0x64` | Key reset | Native XML | `reset_key` |
| `0x65` | Key sync | JS | dedicated handler, see the control-name warning below |
| `0x66` | Beat sync | Inconsistent XML | declared as `script-binding` without a JS function |
| `0x70` / `0x71` | Rewind / Forward | Native XML | `back` / `fwd` |
| `0x4C` / `0x4D` | Loop halve / double | JS | dedicated SHIFT codes |

#### Deck-specific notes

- **Load track**
  - Deck 1: `0x96 / 0x46`
  - Deck 2: `0x96 / 0x47`
  - Deck 3: `0x96 / 0x48`
  - Deck 4: `0x96 / 0x49`

- **Orientation**
  - Decks 1 and 3: `0x16`
  - Decks 2 and 4: `0x18`

- **Beatjump**
  - declared on `0x19` and `0x1A`,
  - but decks 2 to 4 incorrectly reuse `status 0x90`,
  - which creates a collision with deck 1.

### 6.2 Jog wheels

The script implements:

- **touch** through `wheelTouch`:
  - `value 0x7F` → `engine.scratchEnable(...)`
  - `value 0x00` → `engine.scratchDisable(...)`

- **rotation** through `wheelTurn`:
  - if Mixxx is scratching → `engine.scratchTick(...)`
  - otherwise → `engine.setValue(group, "jog", ...)` for pitch bend

The XML declares two rotation inputs (`0x21` and `0x22`), but they call the **same** JS function.  
In practice, the behavior mainly depends on the scratch state set by touch, not on the MIDI number alone.

### 6.3 Tempo

#### Implemented
- 14-bit tempo fader per deck through JS (`rate_msb` + `rate_lsb`)
- Native Mixxx reset on `0x41` (`rate_set_default`)
- Additional JS reset on `0x46` (`rate_reset`)
- Range selector through `RangeSelector` on `0x60`

#### Technical note
The JS properly combines MSB + LSB, but then applies a fixed internal scale of **±8%** in `_updateRate`.  
The `RangeSelector` handler increments `rateRange` numerically with a modulo 4; the exact validity of this logic depends on the way Mixxx interprets that value.  
Because the JS fader calculation remains independent from the selected range, the whole tempo block should be considered **to be validated on real hardware**.

### 6.4 Transport and playback

Per deck:

- `play`
- `cue_default`
- `slip_enabled`
- `reverse` (JS)
- `back` / `fwd`
- `sync_enabled`
- `reset_key`
- `keylock`
- `quantize`
- `LoadSelectedTrack`

### 6.5 Hot Cues

The mapping declares **8 hot cues per deck**:

- **activate**: notes `0x00` to `0x07`
- **clear**: same notes on the next status

Per-deck distribution:

- Deck 1: `0x97` / `0x98`
- Deck 2: `0x99` / `0x9A`
- Deck 3: `0x9B` / `0x9C`
- Deck 4: `0x9D` / `0x9E`

### 6.6 Loops

#### Native controls
- `loop_in`
- `loop_out`
- `reloop_toggle`

#### JS controls
- `LoopHalveShift` on `0x4C`
- `LoopDoubleShift` on `0x4D`
- `reverse` on `0x15`

The JS file also contains older handlers (`loopIn`, `loopOut`, `loopManual`, `autoLoop`, `reloop`), but they are no longer called by the current XML.

### 6.7 Mixer, gain, volume, and EQ

#### Master
- `gain`: `0xB6 / 0x08`
- `crossfader`: `0xB6 / 0x1F` + `0xB6 / 0x3F` (14-bit)
- `headMix`: `0xB6 / 0x0C` + `0xB6 / 0x2C` (14-bit)
- `headGain`: `0xB6 / 0x0D` + `0xB6 / 0x2D` (14-bit)

#### Per deck
- `pregain` MSB: `0x04`
- `pregain` LSB: `0x24`
- `volume` MSB: `0x13`
- `volume` LSB: `0x33` **only on decks 1 and 2**
- EQ High: `0x07` + `0x27`
- EQ Mid: `0x0B` + `0x2B`
- EQ Low: `0x0F` + `0x2F`

#### Precision warning
- The **EQs** are correctly declared as 14-bit.
- The **crossfader** and **headphone mix/gain** are correctly declared as 14-bit.
- **pregain** has an LSB (`0x24`) on all 4 decks, but the MSB (`0x04`) is declared as `normal` rather than `fourteen-bit-msb`.
- The **volume fader** has an LSB (`0x33`) only on decks 1 and 2.

### 6.8 Quick Effects

- QuickEffect enable per deck:
  - Deck 1: `0x97 / 0x16`
  - Deck 2: `0x99 / 0x16`
  - Deck 3: `0x9B / 0x16`
  - Deck 4: `0x9D / 0x16`

- `super1` knob:
  - Deck 1: `0xB6 / 0x17` (7-bit)
  - Deck 2: `0xB6 / 0x18` + `0xB6 / 0x38` (14-bit)
  - Deck 3: `0xB6 / 0x19` + `0xB6 / 0x39` (14-bit)
  - Deck 4: `0xB6 / 0x1A` + `0xB6 / 0x3A` (14-bit)

### 6.9 Effect Units

The XML declares:

- **4 effect units**
- assignment to decks through `group_[ChannelX]_enable`
- assignment to **Master**
- assignment to **Headphone**
- enable controls for **effect slots 1 to 3** on notes `0x50`, `0x51`, `0x52`

#### Note
Some assignments also use `0x44`, `0x45`, `0x46`, `0x47` for Headphone routing, on top of other UI / load / microphone controls declared elsewhere.  
This should be validated on real hardware.

### 6.10 Navigation, library, and UI

#### Library
- Browse knob: `0xB6 / 0x40` → `MoveVertical`
- Focus left: `0x96 / 0x65` → `MoveFocusBackward`
- Focus right: `0x96 / 0x7A` → `MoveFocusForward`
- Maximize library: `0x97 / 0x40`

#### UI
- Show mixer: `0x97 / 0x43`
- Show waveforms: `0x97 / 0x41`
- Show 4 decks: `0x97 / 0x42`
- Show microphone: `0x97 / 0x46`
- Show samplers:
  - `0x90..0x93 / 0x22`
  - `0x97 / 0x45`
- Show effect rack:
  - `[EffectRack1] show` on `0x90 / 0x1E`, `0x91 / 0x1E`, `0x97 / 0x44`
  - `[Skin] show_effectrack` also on `0x91 / 0x1E`
- Show 4 effect units:
  - `0x97 / 0x17`
  - `0x99 / 0x17`
  - `0x9B / 0x17`
  - `0x9D / 0x17`
  - plus `0x92 / 0x1E` and `0x93 / 0x1E`

### 6.11 Samplers

The mapping covers all **8 samplers**, but not in a fully symmetrical way.

#### Samplers 1 to 4
- `cue_gotoandplay`: `0x30` to `0x33`
- `start_stop`: `0x34` to `0x37`
- another `cue_gotoandplay` layer: `0x38` to `0x3B`
- another `start_stop` layer: `0x3C` to `0x3F`
- `beatsync`: `0x70` to `0x73`
- `pfl`: `0x74` to `0x77`

#### Samplers 5 to 8
- `cue_gotoandplay`: `0x30` to `0x33` on `status 0x99`
- `start_stop`: `0x34` to `0x37` on `status 0x99`
- `beatsync`: `0x70` to `0x73`
- `pfl`: `0x74` to `0x77`

#### Special case
- **Sampler 3** has a 14-bit `pregain` on `0x03` + `0x23`.

### 6.12 Recording

- `toggle_recording`: `0x97 / 0x47`

## 7. Outputs / LEDs

The XML declares **28 outputs**, which means **7 per deck**:

- `cue_indicator`
- `play_indicator`
- `pfl`
- `quantize`
- `reverse`
- `keylock`
- `sync_key`

##### Notes
- Play / Cue / Quantize / Reverse / Keylock / PFL LEDs are explicitly declared in XML.
- The JS file also contains an `updateLEDs` function, but it is only used internally.
- `syncKeyHandler` writes `key_sync`, while the XML outputs listen to `sync_key`.  
  This naming mismatch must be verified in Mixxx.

## 8. Detected inconsistencies and collisions

### 8.1 MIDI collisions (`status` + `midino`)

| `status` | `midino` | Detected collision |
|---|---|---|
| `0x90` | `0x19` | beatjump backward declared 4 times for decks 1 to 4 |
| `0x90` | `0x1A` | beatjump forward declared 4 times for decks 1 to 4 |
| `0x97` | `0x13` | Effect Unit 4 CH1 assign / Sampler 3 start_stop |
| `0x99` | `0x13` | Effect Unit 4 CH2 assign / Sampler 7 start_stop |
| `0x9B` | `0x13` | Effect Unit 4 CH3 assign / Sampler 3 start_stop |
| `0x91` | `0x1E` | `EffectRack1.show` / `Skin.show_effectrack` |
| `0x90..0x93` | `0x22` | duplicated `show_samplers` between `[Samplers]` and `[Skin]` |

### 8.2 Functional inconsistencies

1. `beatjump_forward`, `beatjump_backward`, and `beatsync` are declared as `script-binding` without JS functions.
2. Decks 2 to 4 use the wrong `status` for beatjump (`0x90` instead of a deck-specific MIDI channel).
3. `syncKeyHandler` writes `key_sync`, while the XML outputs track `sync_key`.
4. `pregain` mixes a 7-bit declaration with a 14-bit LSB.
5. `volume` only has an LSB on decks 1 and 2.
6. `QuickEffect super1` is 7-bit on deck 1 but 14-bit on decks 2 to 4.
7. The internal XML / JS / documentation versions do not match each other.

## 9. Recommended cleanup

1. Replace the orphan `script-binding` entries (`beatjump_forward`, `beatjump_backward`, `beatsync`) with native XML bindings, or add the missing JS functions.
2. Correct the MIDI statuses for beatjump on decks 2 to 4.
3. Harmonize `key_sync` / `sync_key`.
4. Normalize 14-bit handling for `pregain`, `volume`, and `QuickEffect`.
5. Remove or explicitly document the duplicate UI bindings (`show_samplers`, `show_effectrack`).
6. Remove the unused legacy JS handlers to simplify maintenance.

## 10. Recommended test plan

### Basic tests
- Play / Cue / PFL / Keylock / Quantize
- Volume, crossfader, pregain, EQ
- Browse and track loading
- QuickEffect and Effect Unit activation

### JS-specific tests
- Global SHIFT
- 14-bit tempo (MSB + LSB)
- Rate reset
- Rate range
- Jog touch / scratch / bend
- Reverse hold / SHIFT reverse

### Consistency checks to watch closely
- Beatjump on decks 2 to 4
- Beat sync on all 4 decks
- Key sync LED feedback
- Actual volume precision on decks 3 and 4
- Potential conflicts between samplers and FX assignments

---

**Conclusion**  
The mapping already covers a large part of the DDJ-FLX10, but the audit shows a **mixed state**:  
a substantial XML base, a useful JS core for complex behaviors, and several leftovers or collisions that should be cleaned up to achieve fully coherent behavior and documentation.
