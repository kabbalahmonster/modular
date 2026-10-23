# ◆ MODULAR — a serverless web modular synthesizer & mini-DAW

MODULAR is a patchable synthesizer, drum machine, step sequencer, song arranger and
offline renderer that lives in **one static HTML file** (`index.html`, ~6,900 lines,
no dependencies, no build step, no network calls).

You drag cables from outputs to inputs (or onto knobs, to modulate them), sequence
patterns, arrange them into a song, and bounce the result to WAV. Patches are saved
in the browser, exported as JSON, or **shared as a self-contained URL**: the whole rack
is compressed into the link's `#hash`, so no server is ever involved.

> **Status note:** this README documents the code as it stands in `index.html`. A
> review found several functional bugs, including silent noise sources, share links
> that drop sequencer patterns, and an HTML-injection path. See
> [§ 14 Known issues](#14-known-issues) before relying on those features.

---

## Table of contents

1. [Quick start](#1-quick-start)
2. [Core concepts](#2-core-concepts)
3. [The interface](#3-the-interface)
4. [Mouse & keyboard reference](#4-mouse--keyboard-reference)
5. [Module reference](#5-module-reference)
6. [The sequencer in depth](#6-the-sequencer-in-depth)
7. [Song arranger](#7-song-arranger)
8. [Playing live: QWERTY piano, Web MIDI, MIDI learn](#8-playing-live-qwerty-piano-web-midi-midi-learn)
9. [Saving, library, files & share links](#9-saving-library-files--share-links)
10. [Recording & bouncing audio](#10-recording--bouncing-audio)
11. [Built-in patches](#11-built-in-patches)
12. [Architecture (developer guide)](#12-architecture-developer-guide)
13. [Function reference](#13-function-reference)
14. [Known issues](#14-known-issues)

---

## 1. Quick start

1. Open `index.html` in a modern desktop browser (Chrome/Edge recommended; Firefox
   and Safari work without Web MIDI).
2. Browsers only start audio after a click, so the splash screen asks you to pick a
   starting patch: **Acid Line**, **Drum Machine**, **Ambient Drift** or **Empty Rack**.
   If you opened a share link, an extra **⇩ Open shared patch** button appears.
3. Every patch except *Empty Rack* starts playing after ~350 ms. **Space** toggles play/stop.

**Hosting:** copy `index.html` to any static host (GitHub Pages, S3, Netlify, a USB
stick). Nothing else is required.

**`file://` vs `http(s)://`:** in Chromium, the CRUSH module's AudioWorklet fails to
load from a `file://` page, so the app falls back to a `ScriptProcessorNode`. The
start-up toast reads *"ready · fallback dsp"* instead of *"ready · worklet dsp
online"*. Serving the file over HTTP, even with `python3 -m http.server`, enables the
worklet. With it, CRUSH parameters can be modulated at audio rate.

---

## 2. Core concepts

### Modules, jacks, cables

| Thing | Looks like | Meaning |
|---|---|---|
| **Module** | a glass panel with a coloured header | One instrument/effect/utility. Built from `DEFS[type].build()`. |
| **Signal jack** (`sig`) | round plug | Audio or control voltage (CV). Real Web Audio nodes. |
| **Trigger jack** (`trig`) | square plug | Events: *"note on at time t, velocity v, length d, note n"*. JS callbacks scheduled ahead on the audio clock. |
| **Cable** | animated bezier curve in the source module's colour | Always runs from an **output** to either an **input jack** or a **knob**. |

Connection rules (`canConnect`):

* `trig` out → `trig` in only.
* `sig` out → `sig` in **or** any modulatable knob.
* Exact duplicate cables are refused (the existing one is returned).
* An input may receive several cables; they sum (Web Audio semantics).

### Cable depth (attenuverter)

Every signal cable owns a `GainNode`: its **depth**, from −1 to +1. Negative values
invert the signal.
Audio cables default to **1.0**; modulation cables default to **0.5**.
For trigger cables, the depth is a **velocity scale** (0 to 1).

### Signal conventions

* CV is normalised to roughly **−1 … +1**.
* **Pitch CV: 1.0 = +48 semitones (4 octaves)**, i.e. 1/48 per semitone. A SEQ or
  KEYS *Pitch* output patched into an OSC/POLY *Pitch* knob at depth **1.00** tracks
  1:1.
* Exponential destinations (filter cutoff, oscillator pitch, LFO rate, echo tone,
  reverb cut filters…) are modulated through the node's `detune` AudioParam, in
  **cents**. Sweeps therefore sound even across the whole range.
* Reference pitch: an OSC with Oct = 0, Semi = 0 plays **C4 (MIDI 60, 261.6 Hz)**.
  SEQ/KEYS pitch CV is relative to C4.

### Two kinds of modulation

When you drop a cable on a knob:

| Kind | Used when | How it works | Consequence |
|---|---|---|---|
| **Audio-rate** | the parameter declares `mod.targets` (AudioParams) | the cable's gain node feeds the AudioParam directly, scaled by `mod.scale` | sample-accurate; FM/AM at audio rate possible |
| **Control-rate** | no `mod.targets` | every animation frame the source is read through a tiny analyser "tap", `base + depth × cv × scale` is computed in JS and written via `set()` | ~60 Hz update; used for drum engines, sequencer length/swing, reverb size, echo time, etc. |

A modulated knob shows a **yellow ring** (the possible modulation range) and a
**yellow dot** (the live value), and its label turns yellow. The numeric readout shows
the *modulated* value.

### The transport

A single beat-based clock (`createTransport`) drives everything tempo-related:

* Tempo 20–300 BPM; position is shown as `bar.beat.sixteenth` (4/4 assumed).
* Events (`play`, `stop`, `seek`, `loop`, `tempo`) are broadcast to modules through
  `locate()`, `onStop()` and `setT()` hooks.
* Events are scheduled with a **120 ms look-ahead** by a 22 ms timer (the classic "two
  clocks" pattern). Visuals and control-rate modulation run on `requestAnimationFrame`.
* An optional loop region (owned by the Song panel) wraps sample-accurately.

---

## 3. The interface

```text
┌───────────────────────────── toolbar ──────────────────────────────┐
│ ◆ MODULAR │ ▶ ■ bpm 001.1.1 │ ● bars │ ⌨ MIDI learn │ song bounce │ │
│ patch [▼] save library share │ master ═══ meter │ ↶ ↷ │   ⤢ clear ? │
├──────┬─────────────────────────────────────────────────────────────┤
│ pal- │                                                             │
│ ette │                   stage (pan / zoom canvas)                 │
│      │                                                             │
├──────┴─────────────────────────────────────────────────────────────┤
│ inspector bar (context-sensitive)                                  │
└────────────────────────────────────────────────────────────────────┘
```

### Toolbar

| Control | Function |
|---|---|
| **▶ / ❙❙** | Play / pause (resumes from the current position). |
| **■** | Stop and rewind to bar 1. |
| **bpm** | Type a tempo, or drag the field up/down. Undoable (drags coalesce). |
| **001.1.1** | Transport position (bar.beat.sixteenth). |
| **●** (rec) | Real-time recording of the master output to a 16-bit stereo WAV (§ 10). |
| **bars** | 0 = record until you press ● again; N = auto-stop after N bars. |
| **⌨** | Toggle the computer-keyboard piano (also **K**). |
| **MIDI** | Request Web MIDI access and listen to all inputs (hot-plug aware). |
| **learn** | MIDI-learn mode: click a control, then move a hardware knob. |
| **song** | Open/close the song arranger (also **S**). |
| **bounce** | Offline render dialog (faster than real time, optional stems). |
| **patch ▼** | Built-in racks and patches saved in this browser. |
| **save** | Overwrite the current library patch, or *save as…* if there is none (**Ctrl+S**). |
| **library** | Patch library dialog (**L**). |
| **share** | Encode the rack into a URL and copy it to the clipboard. |
| **master** | Master volume (affects the live output only). |
| **meter** | Post-limiter peak meter. |
| **↶ ↷** | Undo / redo (tooltips name the action). |
| **⤢** | Fit the whole rack in view (**F**). |
| **clear** | Delete everything and leave an empty rack (OUT + SCOPE). |
| **?** | Help overlay (**H**). |

### Palette (left)

One button per module type. **Click** adds the module near the centre of the view;
**drag** places it where you release.
Order: OSC, POLY, KEYS, DRUM, FILTER, CRUSH, ECHO, REVERB, ENV, LFO, SEQ, VCA,
MIXER, SCOPE, OUT.

### Module header buttons

| Button | Action |
|---|---|
| 🎲 | Randomise every knob and select on the module, except those flagged `nornd` (e.g. Oct/Semi/Voices/Steps). Toggles are never randomised. One undo step. |
| ⧉ | Duplicate the module, or the whole selection if this module is part of it. **Shift+click** also clones the cables that cross the selection boundary. |
| ▾ / ▸ | Collapse/expand. Collapsed modules skip drawing, and their cables anchor on the header edge. |
| ✕ | Delete the module (or the selection it belongs to). |

### Inspector bar (bottom)

The bar is context-sensitive:

* **Cable selected:** shows *source → destination* and the depth controls. Signal
  cables get a −1…+1 **depth** slider plus **invert** and **1:1** buttons; trigger
  cables get a 0…1 **velocity scale** slider. **unplug ✕** removes the cable.
* **Several modules selected:** a count, plus *duplicate + cables* and *delete*.
* **One module selected:** its title, description and number of ins/outs/params.
* **Nothing selected:** hints, plus module/cable counts and the sample rate.

### Overlays

Splash, Help, Patch Library and Bounce dialogs. Toast messages appear bottom-centre.

---

## 4. Mouse & keyboard reference

### Mouse

| Target | Gesture | Effect |
|---|---|---|
| Output jack | drag | Start a cable; compatible targets pulse. Drop on an input jack or (for `sig`) a knob. |
| Input jack | drag | Pick up the newest cable on it and re-route it (drop on empty space = unplug). |
| Any jack | right-click | Unplug everything on that jack. |
| Cable | click | Select it and edit its depth in the inspector. |
| Cable | right-click | Cut it. |
| Knob | drag up/down | Change the value (**Shift** = fine ×¼). One undo step per drag. |
| Knob | wheel | Change the value (**Shift** = fine). Consecutive ticks merge into one undo step. |
| Knob | double-click | Reset to default. |
| Knob | **Alt**+drag | Trim the depth of the *newest* modulation cable on it. |
| Knob | right-click | Remove the newest modulation cable on it. |
| Module header | click / drag | Select / move (moving a selected module moves the whole selection). |
| Module header | **Shift/Ctrl/⌘**+click | Toggle the module in the selection. |
| Empty canvas | drag | Pan. Also deselects. |
| Empty canvas | **Shift**+drag | Marquee selection (adds to the current selection). |
| Canvas | wheel | Zoom around the cursor (0.28×–2.2×). |
| Palette icon | click / drag | Add a module. |

### Keyboard

| Key | Action |
|---|---|
| **Space** | Play / pause |
| **Ctrl/⌘+Z**, **Ctrl/⌘+Shift+Z**, **Ctrl/⌘+Y** | Undo, redo, redo |
| **Ctrl/⌘+A** | Select all modules |
| **Ctrl/⌘+C / X / V** | Copy / cut / paste the selection (system clipboard, so it works across tabs; paste lands at the mouse pointer; repeated pastes cascade by 24 px) |
| **Ctrl/⌘+D** | Duplicate the selection (internal cables included) |
| **Ctrl/⌘+Shift+D** | Duplicate, also cloning cables that enter/leave the selection |
| **Ctrl/⌘+S** | Save (see toolbar **save**) |
| **Delete / Backspace** | Delete the selected cable, or else the selected modules |
| **F** | Fit view |
| **H** | Toggle help |
| **L** | Open library |
| **S** | Toggle song arranger |
| **K** | Enable the QWERTY piano |
| **Esc** | Close library/help, disable piano & MIDI learn, deselect |

Keys are ignored while the focus is in a text field, and during an offline render.

**QWERTY piano** (when ⌨ is on): `A W S E D F T G Y H U J K O L P ; '` play
chromatically from C upward (the A key plays the base note, C4 by default).
**Z / X** shift the base octave down/up (C1…C7). While the piano is on, those letters
no longer trigger their shortcuts; use **Esc** to leave.

---

## 5. Module reference

Legend for the parameter tables:

* **Curve** is the knob's travel mapping (`lin`, `exp`, `pow2`, `pow3`).
* **Mod** says whether a cable can be dropped on the knob. `audio` = sample-accurate
  AudioParam modulation; `ctrl` = control-rate (~60 Hz); `–` = not modulatable.
* **Scale** is how many units one full CV (1.0 at depth 1.0) moves the parameter. For
  `cents` destinations the unit is cents.

### 5.1 OSC — multi-wave oscillator (`osc`)

A free-running oscillator with sub-oscillator, pulse-width shaping, two noise colours,
four additive wavetables and slow analogue-style drift.

* **Inputs:** `Trig` (trig). *Currently broken, see issue C2.*
* **Outputs:** `Out` (sig)

Signal flow: `osc → (wave crossfade) ┬ sub ┬ white ┬ pink → Level → Out`. *Pulse* is
a sawtooth plus a DC offset pushed through a hard `tanh` shaper, so moving the offset
(PW) moves the duty cycle.

| Knob | Range | Default | Curve | Mod | Scale | Notes |
|---|---|---|---|---|---|---|
| Oct | −4…+4 | 0 | lin, step 1 | ctrl | 4 | Octave of the base pitch. Not randomised. |
| Semi | −12…+12 | 0 | lin, step 1 | ctrl | 12 | Semitone offset. Not randomised. |
| Pitch | −1200…+1200 ¢ | 0 | lin | audio → `detune` | 4800 ¢ | **Main pitch-CV destination.** Also drives the sub. |
| PW | −0.92…+0.92 | 0 | lin | audio | 0.9 | Pulse width; only audible with *pulse*. |
| Sub | 0…1 | 0 | pow2 | audio | 1 | Square sub-oscillator level. |
| SubOct | 1…2 | 1 | lin, step 1 | – | – | Sub sits 1 or 2 octaves down. |
| FM Amt | 0…4000 Hz | 0 | pow3 | audio | 2000 | Linear-FM depth. **No FM input exists, so this does nothing (issue H4).** |
| Drift | 0…1 | 0.08 | lin | – | – | Random detune (up to ±34 ¢), re-targeted every scheduler tick. |
| Level | 0…1.2 | 0.5 | pow2 | audio | 1 | Output level. |

**Wave:** `sine, triangle, saw, square, pulse, organ, glass, vocal, digi, white, pink`.
`organ/glass/vocal/digi` are `PeriodicWave`s defined in `initTables()`. `white/pink`
are looping 3-second noise buffers (**currently silent, issue C1**).

Visual: a zero-crossing-triggered mini scope.

### 5.2 POLY — 8-voice polyphonic synth (`poly`)

Two oscillators per voice → low-pass filter → amp. One ADSR shapes both the amp and
the filter (via the filter's `detune`).

* **Inputs:** `Notes` (trig, note-aware: each event carries a MIDI note; `dur = null` means "held until release").
* **Outputs:** `Out` (sig)

| Knob | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Wave (select) | saw, square, triangle, sine, organ, glass, vocal, digi | saw | – | Both oscillators. |
| Cutoff | 20 Hz…19 kHz (exp) | 1800 | audio → all voices' filter `detune`, 4800 ¢ | |
| Reso | 0.1…20 | 1 | audio, 8 | |
| EnvAmt | −4800…+4800 ¢ | 2400 | – | Filter envelope depth, scaled by velocity. |
| KeyTrk | 0…1 | 0.4 | – | Cutoff follows the note (1 = 100 %). |
| Attack / Decay / Sustain / Release | 1 ms…6 s / 5 ms…6 s / 0…1 / 5 ms…8 s | 0.01 / 0.3 / 0.6 / 0.35 | – | Read at note-on time. |
| Pitch | ±1200 ¢ | 0 | audio → all 16 oscillator `detune`s, 4800 ¢ | Pitch-CV destination. |
| Detune | 0…100 ¢ | 7 | audio, 50 | Osc 2 offset. |
| Osc2 | 0…1 | 0.5 | audio, 0.5 | Osc 2 level. |
| Level | 0…1.4 | 0.8 | audio, 1 | |
| Voices | 1…8 | 8 | – | Active voice count. Lowering it releases held voices above the limit. |
| Glide | 0…0.5 s | 0 | – | Portamento, **only when Voices = 1**. |
| VelSens | 0…1 | 0.6 | – | Velocity sensitivity (amp and filter envelope). |

**Voice allocation:** a free voice is taken first (the one whose release ended
earliest). Otherwise a voice is stolen: releasing voices before held ones, then the
oldest. A stolen voice fades out in about 12 ms before restarting. Timed notes (from
SEQ) release themselves; held notes (from KEYS) wait for a matching note-off.

**play chord** pad: plays Cmaj7 (C4-E4-G4-B4) for 0.9 s. The visual shows one bar per
voice with its current note name.

### 5.3 KEYS — keyboard / MIDI input (`keys`)

Turns the on-screen keyboard, the QWERTY piano and Web MIDI into patchable signals.

* **Outputs:**
  * `Gate` (trig) — mono, legato-aware: held while *any* key is down.
  * `Notes` (trig) — every note-on/off with its note number. Patch this into POLY.
  * `Pitch` (sig) — `(note − 60) / 48`. Last-note priority; falls back to the previous key on release.
  * `Vel` (sig) — the last velocity (0…1).

| Control | Range | Default | Notes |
|---|---|---|---|
| Ch (select) | all, 1…16 | all | MIDI channel filter. |
| Listen (toggle) | – | on | Whether QWERTY/MIDI notes reach this module. |
| Legato (toggle) | – | off | Overlapping notes only change pitch; no re-gate. |
| Oct | −3…+3 | 0 | Transposes all incoming notes. |
| Glide | 0…0.5 s | 0 | Portamento on the Pitch CV. |

The on-screen keyboard covers C4–B5 and supports glissando (drag across keys).

### 5.4 DRUM — synthesized percussion voice (`drum`)

Each hit builds fresh oscillators/noise sources from the knob values *at trigger time*.
Drum knobs are control-rate, so modulation affects the next hit.

* **Inputs:** `Trig` (trig, velocity-sensitive up to 1.3)
* **Outputs:** `Out` (sig)

| Control | Range | Default | Mod | Meaning (per engine) |
|---|---|---|---|---|
| Engine | kick, snare, hat, clap, tom, rim, cowbell, zap | kick | – | Synthesis recipe. |
| Tune | ±24 st | 0 | ctrl | Pitch of tonal parts. |
| Decay | 0…1 | 0.4 | ctrl | Length. |
| Snap | 0…1 | 0.4 | ctrl | Attack click / pitch-sweep amount / band-pass Q. |
| Tone | 0…1 | 0.5 | ctrl | Brightness; sine↔triangle (kick), sine↔saw (zap). |
| Noise | 0…1 | 0.3 | ctrl | Amount of the noise component. |
| Drive | 1…12 | 1 | ctrl | Pre-gain into a `tanh` shaper. |
| Level | 0…1.5 | 0.85 | audio | Output level. |

Engine recipes:

* **kick:** a sine/triangle sweep from up to 8× the base frequency down to 48 Hz × tune, plus a high-passed noise click.
* **snare:** two triangle tones plus band-passed noise.
* **hat:** six detuned squares (808-style ratios) plus high-passed noise.
* **clap:** three 11 ms noise bursts plus a tail.
* **tom:** a sine sweep plus noise.
* **rim:** two short squares plus noise.
* **cowbell:** two squares at a 1:1.48 ratio.
* **zap:** a long downward sweep.

The **trigger** pad fires a manual hit. (Noise-based parts are currently silent; see
issue C1. **clap** is entirely noise-based and therefore produces no sound.)

### 5.5 FILTER — multimode 12/24 dB filter with drive (`filter`)

`in → drive (tanh) → biquad → [biquad] → wet/dry → out`

| Control | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Type | lowpass, highpass, bandpass, notch, allpass, peaking, lowshelf, highshelf | lowpass | – | |
| 24dB (toggle) | – | off | – | Cascades a second biquad (Q × 0.7). |
| Cutoff | 20 Hz…19 kHz (exp) | 1400 | audio → `detune`, 4800 ¢ | |
| Reso | 0.1…22 | 2 | audio, 10 | |
| Drive | 0.3…14 (exp) | 1 | audio → pre-gain, 6 | Post-gain compensates by `1/drive^0.6` (not modulated). |
| Shelf | ±24 dB | 0 | audio, 18 | Only audible for peaking/shelf types. |
| Mix | 0…1 | 1 | audio (wet only), 1 | Dry/wet. |
| Out | 0…2 | 1 | audio, 1 | Output level. |

Visual: the live spectrum behind the computed magnitude response (squared in 24 dB
mode), with a dashed marker at the modulated cutoff.

### 5.6 CRUSH — bit crusher / decimator / wavefolder (`crush`)

Per sample: `tanh` saturation → sine wavefolder → additive noise ("grit") → bit
quantisation → sample-and-hold decimation → dry/wet. Runs in an AudioWorklet (a-rate
params) or a ScriptProcessor fallback (control-rate params).

| Knob | Range | Default | Mod (worklet) | Scale |
|---|---|---|---|---|
| Bits | 1…16 (continuous) | 8 | audio | 7 |
| Rate | 1…90 (hold factor, exp) | 1 | audio | 30 |
| Drive | 0.3…16 (exp) | 1 | audio | 6 |
| Fold | 0…1 | 0 | audio | 0.5 |
| Grit | 0…1 | 0 | audio | 0.5 |
| Mix | 0…1 | 1 | audio | 0.5 |

Visual: the static transfer curve (a staircase) overlaid with the live output
waveform.

### 5.7 ECHO — ping-pong delay (`echo`)

Two 4-second delay lines with cross-feedback through HP → LP filters, panned hard
L/R. An internal LFO ("wobble") modulates both delay times for tape flutter/chorus.

| Control | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Sync | free, 16/1 … 1/32 (incl. triplets) | 1/8 | – | Tempo-synced delay time. Capped at 3.9 s. |
| Time | 5 ms…2 s (exp) | 0.3 s | ctrl, 0.5 | Only used when Sync = free. |
| R ratio | 0.25…2 | 1 | ctrl, 0.75 | Right delay = left × ratio. |
| Fb | 0…1.15 | 0.45 | audio, 0.5 | Feedback. **Above 1.0 it self-oscillates and grows without bound (issue M6).** |
| Tone | 300 Hz…18 kHz | 5200 | audio → `detune`, 3600 ¢ | Feedback low-pass. |
| HP | 20 Hz…2 kHz | 120 | audio → `detune`, 2400 ¢ | Feedback high-pass. |
| Wobble | 0…1 | 0 | audio, 0.01 | Flutter depth (up to 4 ms). |
| W.Rate | 0.05…8 Hz | 0.4 | audio, 4 | Flutter rate. |
| Mix | 0…1 | 0.35 | audio (wet), 0.5 | Dry is reduced to `1 − 0.35·mix`. |
| Freeze (toggle) | – | off | – | Closes the input and sets feedback to 1.0: an infinite loop. |

### 5.8 REVERB — generated convolution reverb (`rev`)

A `ConvolverNode` whose stereo impulse response is synthesised in JS: filtered noise,
a power-law decay and sparse early reflections, normalised. Changes to Size, Decay,
Damp and Diffuse rebuild the IR after a 140 ms debounce.

| Knob | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Size | 0.15…8 s | 2.2 | ctrl (**ineffective, issue M2**) | IR length. |
| Decay | 0…1 | 0.4 | ctrl (ineffective) | Decay curve exponent. |
| Damp | 0…1 | 0.45 | ctrl (ineffective) | High-frequency damping over time. |
| Diffuse | 0…1 | 0.3 | ctrl (ineffective) | Early-reflection strength. |
| PreDly | 0…0.25 s | 0.015 | audio, 0.1 | |
| LowCut | 20 Hz…1.2 kHz | 120 | audio → `detune`, 2400 ¢ | Pre-convolution HP. |
| HiCut | 800 Hz…19 kHz | 9000 | audio → `detune`, 3600 ¢ | Pre-convolution LP. |
| Mix | 0…1 | 0.3 | audio, 0.5 | Wet = 1.3·mix; dry = 1 − 0.5·mix. |

Visual: the IR envelope.

### 5.9 ENV — DAHDSR envelope (`env`)

A `ConstantSourceNode` automated through Delay → Attack → Hold → Decay → Sustain →
Release, then multiplied by **Depth** and offset by **Bias**. The output is an
ordinary signal, so it can modulate knobs or be listened to as audio. It is always
tapped, so its live level is available.

* **Inputs:** `Trig` (trig). A timed event (`dur` given) releases after `dur`. A held event (`dur = null`) should release on note-off, **but ENV has no release handler (issue C3).**
* **Outputs:** `Env` (sig)

| Control | Range | Default | Mod |
|---|---|---|---|
| Delay | 0…2 s | 0 | ctrl |
| Attack | 1 ms…8 s | 10 ms | ctrl |
| Hold | 0…2 s | 0 | ctrl |
| Decay | 2 ms…8 s | 0.25 s | ctrl |
| Sustain | 0…1 | 0.5 | ctrl |
| Release | 2 ms…12 s | 0.4 s | ctrl |
| Depth | −1…+1 | 1 | audio |
| Bias | −1…+1 | 0 | audio |
| Curve (select) | exp, lin | exp | – |
| Loop (toggle) | – | off | – |
| Vel | 0…1 | 0.3 | ctrl |

* Stage times are read at trigger time (modulation included).
* Every trigger restarts from 0.
* **Vel** sets velocity sensitivity: `peak = 1 − vel + vel × velocity`.
* **Loop** re-fires the envelope free-running with period `dly+a+hold+d+r`.
* The **gate** pad behaves like a held key: press = attack, release = release.

### 5.10 LFO — low-frequency oscillator / random source (`lfo`)

* **Inputs:** `Retrig` (trig). Restarts the periodic oscillator's phase. Random modes ignore it.
* **Outputs:** `LFO` (sig), always tapped.

| Control | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Wave | sine, triangle, saw, square, s&h, random, drift | sine | – | First four use an `OscillatorNode`. `s&h` = stepped random values; `random` = linear ramps between them; `drift` = smooth exponential glide. |
| Sync | free, 16/1 … 1/32 | free | – | Rate from the tempo (cycle = division length). |
| Rate | 0.002…200 Hz (exp) | 2 Hz | audio → `detune`, 4800 ¢ (periodic waves); ctrl (random waves) | Displayed as a period in seconds below 1 Hz. Audio-rate up to 200 Hz. |
| Depth | −1…+1 | 1 | audio | |
| Bias | −1…+1 | 0 | audio | |
| Smooth | 0…1 | 0 | ctrl | Low-pass on the output: off at 0, then 40 Hz → 0.4 Hz cutoff; turns s&h into glides. |

### 5.11 SEQ — step sequencer (`seq`)

See [§ 6](#6-the-sequencer-in-depth).

* **Inputs:** `Reset` (trig)
* **Outputs:** `Gate` (trig, carries the note number), `Pitch` (sig, semitones/48), `Vel` (sig)

### 5.12 VCA — amplifier + panner (`vca`)

| Knob | Range | Default | Mod |
|---|---|---|---|
| Gain | 0…1.4 (pow2) | **0** | audio |
| Pan | −1…+1 | 0 | audio |

Gain defaults to **0**: patch an envelope into the Gain knob to open it.

### 5.13 MIXER — 4-channel mixer (`mix`)

* **Inputs:** `In1`…`In4`
* **Outputs:** `Out`

| Knob | Range | Default | Mod |
|---|---|---|---|
| Lvl1 … Lvl4 | 0…1.5 | 0.8 / 0.6 / 0.6 / 0.6 | audio |
| Pan1 … Pan4 | −1…+1 | 0 | audio |
| Master | 0…1.5 | 0.9 | audio |

Visual: per-channel peak bars.

### 5.14 SCOPE — oscilloscope / spectrum / XY (`scope`)

* **Inputs:** `In A` (stereo, split L/R), `In B` (Y axis in XY mode)
* **Outputs:** `Thru` (passes In A unchanged)

| Control | Range | Default | Notes |
|---|---|---|---|
| Mode | scope, spectrum, xy, stack | scope | *stack* = scope on top, spectrum below. |
| Size | S, M, L | M | 130 / 210 / 300 px tall. |
| Freeze | toggle | off | Stops redrawing. |
| Time | 0.3…90 ms | 12 | Window width. |
| Gain | 0.1…10 | 1 | Display gain. |
| Trig | −1…+1 | 0 | Rising-edge trigger level. |
| Persist | 0…0.97 | 0.55 | Phosphor trail (partial clear of an offscreen canvas). |

The right channel is drawn (pink) only when it differs from the left. The spectrum is
log-ish with 120 bars and peak hold. XY uses In A left as X and In B (or In A right)
as Y.

### 5.15 OUT — output to the speakers (`out`)

Every audible patch needs at least one OUT. Chain:
`In → Level → Pan → [per-module limiter] → master bus`.

| Control | Range | Default | Mod |
|---|---|---|---|
| Level | 0…1.5 | 0.8 | audio |
| Pan | −1…+1 | 0 | audio |
| Limiter | toggle | on | – (compressor: −6 dB threshold, 14:1) |

`Thru` outputs the post-pan, **pre-limiter** signal (handy for a SCOPE). The visual is
a peak meter with a clip flag.

**Master bus** (global): `bus gain (master slider, 0.72 default) → 12 Hz DC-blocking
high-pass → compressor (−4 dB, 12:1, 3 ms attack) → analyser/meter → speakers`.

---

## 6. The sequencer in depth

### Step model

Each pattern has 32 steps:
`{ on, note 0…24, vel 0…1, gate 0.03…1, prob 0…1, ratch 1…4, slide 0|1 }`.
Each SEQ holds **8 patterns (A–H)**.

### Lanes and editing

Pick a lane (**ON / PITCH / VEL / GATE / PROB / RATCH / SLIDE**), then click or drag
across the grid:

| Lane | Vertical position sets | Notes |
|---|---|---|
| on | – | The first click toggles the step; dragging turns steps on. |
| pitch | note 0…24 (two octaves above the root, before scale quantisation) | Also turns the step on. |
| vel | velocity | Also turns the step on. |
| gate | gate length as a fraction of the step | Also turns the step on. |
| prob | trigger probability | Also turns the step on. |
| ratch | 1–4 repeats inside the step | Also turns the step on. |
| slide | on (upper half) / off | Glides pitch CV *into* this step. |

**Shift+drag** turns steps off in any lane. Each drag is one undo step.

Grid display: bar height = pitch, brightness = velocity, yellow marker = the active
lane's value, dots = ratchets, underline = slide. The red column is the playhead; a
red outline marks the step-record cursor.

### Controls

| Control | Range | Default | Mod | Meaning |
|---|---|---|---|---|
| Div | 16/1 … 1/32 | 1/16 | – | Step length. |
| Dir | fwd, rev, pingpong, random, brownian | fwd | – | Play order. Brownian moves ±1–2 steps. |
| Scale | Chromatic, Major, Minor, Dorian, Phrygian, Lydian, Mixolyd, HarmMin, PentMin, PentMaj, Blues, Whole | PentMin | – | Notes snap to the nearest degree. |
| Root | C…B | C | – | Scale root (also transposes). |
| Steps | 1…32 | 16 | ctrl | Pattern length. |
| Swing | 0…0.7 | 0 | ctrl | Even steps are lengthened by (1+sw), odd steps shortened by (1−sw). |
| Oct | −3…+3 | 0 | ctrl | Octave transpose. |
| Transp | ±12 st | 0 | ctrl | Semitone transpose (applied after the scale). |
| Human | 0…1 | 0 | ctrl | Random velocity reduction (up to 50 %) and −6…+12 ms timing jitter. |
| Glide | 5 ms…0.4 s | 0.06 | ctrl | Slide time. |
| Gate× | 0.1…2 | 1 | ctrl | Global gate-length multiplier. |

### Outputs per fired step

1. `n = quantise(note + root) + transpose + 12·oct`
2. **Pitch** CV = `n / 48`: a step, or an exponential glide if *slide* is set.
3. **Vel** CV = `velocity` (after humanise).
4. **Gate** fires `ratch` times across the step. Each sub-gate's length is
   `gate × Gate× × stepLength / ratch`, clamped to 10 ms–4 s, and it carries
   MIDI note `60 + n` (so a SEQ can drive POLY directly).

### Pattern tools

| Button | Action |
|---|---|
| rnd | Random notes, about 72 % of steps on, random velocities (over the current length). |
| euc | Euclidean-style fill with a random density of 25–60 %. |
| clr | Reset the pattern to defaults (a four-on-the-floor of `on` steps). |
| ◀ ▶ | Rotate the pattern left/right. |
| inv | Invert pitches (`24 − note`). |
| copy / paste | Pattern clipboard shared by all SEQs. |
| ● rec | Arm recording (see below). |

### Patterns A–H

* Click a letter **while playing** to *queue* the switch. It happens when the pattern
  restarts, and the letter blinks while queued.
* **Shift+click** switches immediately. When stopped, or in song mode, the switch is
  always immediate.
* Letters with an underline contain non-default data.

### Recording

With **● rec** armed, notes from the **QWERTY piano or Web MIDI** are written into the
current pattern:

* **Stopped = step record.** Each note fills the next step and advances the cursor.
* **Playing = live record.** The note snaps to the nearest step (a note played more
  than half a step late, in forward mode, goes to the next step).

Incoming notes are converted back to scale degrees relative to Root/Transp/Oct and
folded into the 0…24 range. Notes played on a KEYS module's *on-screen* keyboard are
not recorded (see issue L9).

### Transport locking

When playback starts, seeks or loops, a forward-playing SEQ derives its step from the
transport's beat position, so several SEQs always stay in phase. The **Reset** input
jumps back to step 0.

---

## 7. Song arranger

Open it with **song** or **S**. It shows one lane per SEQ module and a bar ruler.

| Control | Function |
|---|---|
| **song mode** | When ON, clips decide which pattern each SEQ plays. A SEQ that has clips is **silent in the gaps between them**. SEQs with no clips play normally. |
| **loop** | Wrap the transport at the song length. This is independent of song mode. If song mode is on and loop is off, playback stops at the end ("song finished"). |
| **bars** | Song length, 1–256. Shortening it trims clips. |
| **brush A–H** | Pattern used for new clips and for click-to-paint. |
| **zoom** | 16–120 px per bar. |
| Click an empty lane | Add a 1-bar clip with the brush pattern. |
| Drag a clip | Move it (no overlaps allowed). |
| Drag a clip's right edge | Resize it. |
| Click a clip | Repaint it with the brush pattern. |
| Right-click a clip | Delete it. |
| Click the ruler | Seek to that bar. |

When song mode drives a SEQ, its pattern view follows the song. All arranger edits are
undoable, and the song is saved with the patch and included in share links.

---

## 8. Playing live: QWERTY piano, Web MIDI, MIDI learn

All note input goes through one hub (`noteInput`). Notes reach every **KEYS** module
that has **Listen** on and a matching **Ch**. They are also written into every SEQ with
**● rec** armed.

* **QWERTY piano:** see [§ 4](#4-mouse--keyboard-reference). Key-up always releases
  the note that key started, even if you changed octave in between. Losing window
  focus releases all notes (panic).
* **Web MIDI:** press **MIDI** (Chrome/Edge; Firefox with permission). All inputs are
  bound, including devices plugged in later. The app handles note-on, note-off
  (including note-on with velocity 0), CC, and CC 120/123 (all sound/notes off → panic).
* **MIDI learn:** press **learn**, click a knob/select/toggle (it blinks), then move a
  hardware control. Knobs map CC 0–127 across their travel (honouring the curve);
  toggles switch at 64; selects divide the range evenly among their options.
  Right-click a control in learn mode to clear its mapping. Mappings are saved with
  the patch (`mc` field) and survive copy/paste.

---

## 9. Saving, library, files & share links

### Library (browser storage)

Patches are stored in `localStorage["mod_patches"]` as `{ name: patch }`. Open the
library with **library** or **L**:

* **Built-in:** load a factory rack.
* **Saved in this browser:** search, then per row: load (or click the name),
  ⤓ overwrite with the current rack, ✎ rename, ⧉ duplicate, ↧ export as `.json`,
  ✕ delete.
* **＋ save current as…**, **↥ import…** (one or many `.json` files, single patches
  or bundles), **↧ export all** (a `modular-library.json` bundle), **↺ restore last
  session**, **clear library**.

**Autosave:** every 20 s the working rack is written to `localStorage["mod_session"]`.
Restore it from the library dialog.

### Patch file format (schema v2)

```jsonc
{
  "v": 2, "kind": "patch", "bpm": 124, "name": "my patch", "date": 1700000000000,
  "mods": [
    { "id": "m3", "t": "osc", "n": "OSC 1", "x": 560, "y": 40, "c": 0,
      "p": { "wave": "saw", "lvl": 0.55 },          // every parameter's base value
      "s": null,                                     // def.save() state (SEQ patterns…)
      "mc": { "lvl": [null, 74] } }                  // MIDI-learn: [channel|null, cc]
  ],
  "cs": [ { "f": ["m1","pitch"], "t": ["m3","#fine"], "a": 1 } ],   // '#param' = modulation
  "song": { "on": false, "loop": false, "bars": 8, "tracks": { "m1": [ {"s":0,"l":2,"b":0} ] } }
}
```

* v1 files (no `n`/`kind`) are migrated automatically.
* Unknown module types are skipped.
* Parameter values are validated: selects must match an option, and numbers must be
  finite and are clamped.

### Share links

**share** encodes the whole rack into the URL fragment and copies it:

```text
https://your.host/index.html#p=z.<base64url(deflate-raw(JSON))>
                              #p=j.<base64url(JSON)>      (browsers without CompressionStream)
```

The packed JSON is compact:

```text
{ v, b: bpm,
  m: [ [type, x, y, collapsed, {paramId: value  (non-defaults only, 4 sig. digits)}, state] … ],
  c: [ [fromIdx, fromJack, toIdx, jack | '#param', depth] … ],
  sg: { o: songOn, l: loop, n: bars, t: [ [modIdx, [[start, len, bank] …]] … ] } }
```

SEQ patterns use a columnar codec: `on`/`slide` as 32-bit masks, notes as a base-36
string, and other lanes only when they differ from the defaults. *(A version mismatch
currently breaks this codec, so patterns are lost; see issue C4.)*

* Because the fragment never reaches a server, the patch is never uploaded anywhere.
* On first load, a link adds **⇩ Open shared patch** to the splash. Pasting a link
  into an already-open tab loads it immediately as an undoable step.
* Safety limits: 2 MB decompressed, 200 modules, 2000 cables.
* Links longer than 8000 characters trigger a warning, because some apps truncate them.
* In the console, `await SHARE.report()` shows what each part of the patch costs in bytes.

### Clipboard fragments

Copy/cut puts a JSON **fragment** (the same schema, `kind: "fragment"`, with internal
cables only) on the system clipboard, so you can paste between tabs or save snippets
in a text file.

---

## 10. Recording & bouncing audio

### ● Real-time recorder

* Taps the post-limiter master and records until you press ● again, or for **bars**
  bars measured from the moment you press it.
* Writes a 16-bit stereo WAV named `modular-<timestamp>.wav`.
* Uses a `ScriptProcessorNode`.

### Bounce (offline render)

**bounce** opens a dialog:

| Field | Meaning |
|---|---|
| length (bars) | Defaults to the song length. 1–512 bars; the total is capped at 600 s. |
| tail (seconds) | 0–30 s of extra render time for reverb/delay tails. |
| format | 16-bit PCM, 24-bit PCM, or 32-bit float WAV. |
| stems | Also render one stereo file per OUT module (up to 15), zipped with the master. |

How it works:

1. The live rack is serialised.
2. A hidden copy is built on an `OfflineAudioContext` (`withHeadlessRack`), so the
   visible rack is never touched.
3. The live context is suspended.
4. The offline context is suspended every 1024 frames, so that control-rate
   modulation and the event scheduler run exactly as they would live.

If song mode is on, the arrangement is used; looping is disabled offline. The ZIP
writer is a built-in store-only implementation.

---

## 11. Built-in patches

| Name | Contents |
|---|---|
| **init / Empty Rack** | OUT → SCOPE. |
| **acid** (128 BPM) | SEQ (PentMin, swing, slides, a ratchet) → OSC saw + sub → FILTER 24 dB (resonant; modulated by ENV + a slow LFO) → CRUSH → VCA (ENV + velocity) → ECHO 1/8T → OUT → SCOPE. |
| **drums** (122 BPM) | Three SEQ→DRUM lanes (kick, snare, hat; velocity → level) → MIXER → REVERB → OUT → SCOPE in *stack* mode. |
| **ambient** (68 BPM) | Slow Lydian SEQ with slides → two OSCs (glass + organ, detuned; random LFO drift) → FILTER (LFO + ENV) → VCA (long ENV) → ECHO → REVERB → OUT → SCOPE in XY mode. |

Built-ins are written as code (`P_init`, `P_acid`, `P_drums`, `P_ambient`) rather
than JSON.

---

## 12. Architecture (developer guide)

### File layout

`index.html` contains three blocks:

1. **`<style>`:** CSS in 10 labelled sections (tokens, layout, toolbar, palette,
   stage/cables, module chrome, knobs, inspector, overlays, song/export). Each module
   sets its own colour via `--mc`.
2. **`<body>`:** the static skeleton: toolbar, palette, stage (`#scene` with the SVG
   `#wires` layer), inspector, toast area, and overlays (splash, export, library,
   help).
3. **`<script>`:** the application, in numbered sections:

| § | Section | Responsibility |
|---|---|---|
| 1 | Tiny helpers | `h`, `$`, `clamp`, `lerp`, `rnd`, `toast`; global context-menu suppression |
| 2 | Audio engine core | `createEngine` (context, master chain, noise, wavetables, worklet), `ENGINE` singleton, crusher worklet source |
| 3 | Music theory & clock | `NOTE`, `mtof`, `noteName`, `SCALES`, `quantN`, `DIVS`, `createTransport`, `fmtPos` |
| 4 | Parameter system | value↔knob mapping, formatting, class `P`, widget factories |
| 5 | Patch state & view | registries (`modules`, `conns`, `liveParams`, `tapped`), transport↔module wiring, rack context, pan/zoom, **undo/redo history** |
| 6 | Module factory | `addModule`, `removeModule`, selection, group drag |
| 7 | Jacks, cables & routing | taps, connect/disconnect, cable SVG, cable drag, trigger protocol |
| 8 | Inspector bar | `showInsp` |
| 9 | Stage navigation, clipboard, duplicate | pan, marquee, zoom, fit, copy/cut/paste, duplicate |
| 10 | Scheduler | `readTaps`, `pumpControl`, `pumpSched`, 22 ms interval |
| 11 | Module definitions | `DEFS.osc … DEFS.keys` |
| 12 | Palette | buttons from `ORDER` |
| 13 | Render loop | rAF: control-rate mods, visible-module drawing, cables, meter, song panel |
| 14 | Presets & serialisation | fragments, schema/migrations, import/export, built-in racks |
| 15 | Patch library / 15.5 Share links | localStorage UI, autosave, `SHARE` codec |
| 16 | Recorder | real-time WAV capture |
| 17 | Toolbar & keyboard | global buttons, shortcuts |
| 19 | Note input hub | QWERTY, Web MIDI, MIDI learn |
| 20 | Song | model + arranger panel |
| 21 | Offline render | headless rack, `renderOffline`, WAV/ZIP encoders, export dialog |
| 18 | Boot | splash → `ENGINE.init` → load patch |

### Engine

`createEngine(ac)` wraps an `AudioContext` *or* an `OfflineAudioContext`, and exposes:

* `ac`, `now()`
* `setAP(param, value, tc)`: a click-free `setTargetAtTime`.
* `cah(param, t)`: cancel-and-hold.
* `bus`/`masterGain`, `masterAna`
* `noise {white, pink}`, `waves` (PeriodicWaves), `hasWorklet`
* `transport`
* `init({workletSrc})`: async; creates the noise buffers and wavetables, and registers
  the worklet from a Blob URL.

Modules **must** take everything from `m.ctx` (the engine they were built on) rather
than from globals. This is what lets the same `build()` run on the offline engine.

### Module contract

`addModule(type, x, y, id?)` creates the frame and calls `DEFS[type].build(m)`.

The module object `m` provides:

* **Identity and geometry:** `id, type, def, ctx, x, y, w, hh`
* **DOM:** `el, head, body, inCol, center, outCol, ctrls`
* **Parameters:** `params` (by id), `plist` (in order)
* **Jacks and nodes:** `ins, outs, nodes`
* **Free-form state:** `state`

Helpers on `m`:

* `m.param(def)`, `m.knobs([defs], host?)`
* `m.addIn(def)`, `m.addOut(def)`: `{id, label, kind:'sig'|'trig', node?, onTrig?, onRel?}`
* `m.viz(height)` → a HiDPI 2D context (`ctx.canvas.W/H` in CSS pixels)
* `m.label(text)`, `m.node(audioNode)` (registers the node for teardown),
  `m.setCollapsed(flag)`

Optional hooks a module sets:

| Hook | Called | Purpose |
|---|---|---|
| `m.draw(t)` | each frame, only if visible and not collapsed | visuals |
| `m.sched(now, ahead)` | every 22 ms (live) or every 1024 frames (offline) | schedule events in `[now, ahead]` |
| `m.locate(beat, time)` | transport play/seek/loop | phase-lock |
| `m.reset(time)` | legacy fallback for `locate` | |
| `m.onStop(rewind)` | transport stop | (no module implements it yet) |
| `m.setT()` | tempo change | recompute tempo-synced times |
| `m.trigNow(t, vel, dur)` | – | manual trigger convenience |

Optional `def` hooks:

| Hook | Purpose |
|---|---|
| `save(m)` / `load(m, s)` | persistent state (patch files, clipboard) |
| `snap(m)` | undo-relevant subset of the state (falls back to `save`) |
| `pack(m)` / `unpack(p)` | compact share-link codec |
| `free(m)` | custom teardown |

### Parameter definitions

```js
{ k: 'knob'|'select'|'toggle', id, label,
  min, max, def, step?, curve: 'lin'|'exp'|'pow2'|'pow3', unit?, fmt?,
  opts?,                    // selects
  cents?: true,             // modulation is exponential (base · 2^(cv·scale/1200))
  nornd?: true,             // excluded from 🎲
  mod?: { targets?: [AudioParam…], scale },   // omit = not modulatable
  modMap?: (base, sum) => value,             // custom modulation mapper
  set: v => { … } }         // push the value into the audio graph
```

### Trigger / note protocol

* `onTrig(t, vel, dur, note)`: `t` is audio time; `vel` is already scaled by cable
  depth; `dur` is in seconds, or `null` for "held until release"; `note` is a MIDI
  note number, or `undefined`.
* `onRel(t, note)` ends a held note.
* Sources: SEQ Gate (timed); KEYS Gate/Notes (held).
* Receivers: ENV, POLY, DRUM (ignores note and duration), LFO Retrig, SEQ Reset.
* Helpers: `fireTrig(jack, …)` and `fireRel(jack, …)` walk `jack.conns`.

### Undo/redo

`history` holds command objects `{label, undo(), redo(), key?, merge?}`:

* Maximum depth 300. Commands with the same `key` merge within 600 ms (wheel, slider
  and BPM drags).
* `begin/end/run` group commands into one composite step; `mute` suppresses recording.
* `replace(label, fn)` snapshots the whole rack before and after, and is used for
  loading patches.
* Commands refer to modules **by id** (never by object), because undo re-creates
  modules with the same ids. Ids are never reused (`idc` only grows).

Recorders: `recAddModule`, `recRemoveModule`, `recCable`, `recAmount`, `recParam`,
`recMove`, `recCollapse`, `stateEditor(m).commit()`, `userTempo`, `songCommit`.

### Rendering and timing

* **Audio events:** a `setInterval(22 ms)` loop calls `m.sched(t, t+0.12)` on every
  module and handles seamless loop wrapping (`loopCrossing`/`wrapLoop`).
* **Visuals and control-rate modulation:** the `requestAnimationFrame` loop (capped at
  ~66 fps) reads the taps, applies control-rate params, draws only on-screen modules,
  re-lays cables when `wiresDirty`, and updates the meter, position readout and song
  panel.

### Adding a module

1. Copy an entry in § 11 (e.g. `DEFS.vca`). Give it a unique key, `name`, `glyph`,
   `color`, `w` and `desc`.
2. In `build(m)`, destructure what you need from `m.ctx`. Create nodes via
   `m.node(...)`, register jacks with `m.addIn`/`m.addOut` and parameters with
   `m.knobs`/`m.param`, and optionally set `m.draw`, `m.sched`, `m.locate` and
   `m.setT`.
3. If it has non-parameter state, add `save`/`load` (and `pack`/`unpack` for compact
   share links).
4. Add the key to `ORDER` (§ 12) so it appears in the palette.

---

## 13. Function reference

Line numbers are approximate (they refer to the current `index.html`).

### § 1 Helpers

| Function | Description |
|---|---|
| `h(tag, cls, html)` | Create an element; `html` is assigned via **innerHTML**. |
| `$(sel)` | `document.querySelector`. |
| `clamp(v, a, b)`, `lerp(a, b, t)` | Maths. |
| `rnd(max=1, min=0)` | Random float in [min, max). Note the reversed argument order. |
| `toast(text)` | Transient status message (innerHTML). |

### § 2 Engine

| Function | Description |
|---|---|
| `createEngine(ac)` | Build an engine around a (possibly offline) context: master chain, `setAP`, `cah`, transport and async `init`. |
| `engine.init({workletSrc})` | Resume the context; create noise buffers and wavetables; register the crusher worklet. |
| `mkNoise(ac, pink)` | Create a 3 s mono noise buffer. **Body missing; returns silence (issue C1).** |
| `mkWave(ac, table, name, harmonics)` | Build a `PeriodicWave` from a cosine harmonic list. |
| `initTables(ac, table)` | Define the `organ`, `glass`, `vocal` and `digi` wavetables. |
| `setPlay(p)` | Legacy play/stop wrapper (unused). |
| `clock` | Legacy facade: `clock.bpm` get/set, `clock.playing`. |
| `CRUSH_SRC` | AudioWorkletProcessor source for `crusher`. |

### § 3 Music & transport

| Function | Description |
|---|---|
| `noteName(m)` | MIDI number → `"C4"`. |
| `mtof(m)` | MIDI number → Hz (A4 = 440). |
| `quantN(n, scale, root)` | Snap semitone `n` to the nearest scale degree relative to `root`. |
| `createTransport(engine)` | Beat clock with `on/emit`, `beatSec`, `beatAt`, `timeAt`, `setTempo`, `play`, `stop`, `seek`, `setLoop`, `loopCrossing` and `wrapLoop`. Starts 80 ms ahead for clean scheduling. |
| `fmtPos(beat)` | Beat → `"003.2.4"`. |

### § 4 Parameters

| Function | Description |
|---|---|
| `p2n(p, v)` / `n2p(p, n)` | Parameter value ↔ 0…1 knob position for each curve; `n2p` quantises to `step`. |
| `fmtv(p, v)` | Readout formatting by unit (Hz/k, ms/s, %, ±int). |
| `class P` | A parameter. `modSum()` (Σ depth × cv), `modRange()`, `map(base, sum)` (linear/cents/custom), `effective()`, `apply()`, `setValue(v, silent)`, `refreshMod()` (registers control-rate params and creates source taps), `render()` (knob arc, pointer, mod ring, live dot). |
| `arcP(cx, cy, r, a0, a1)` | SVG arc path. |
| `knobEl(p)` | Knob widget with drag, wheel, double-click, Alt-drag and right-click behaviours. |
| `selEl(p)`, `tgEl(p)` | Select and toggle widgets (undoable). |

### § 5 State, view, history

| Function | Description |
|---|---|
| `locateModule(m, beat, time)` | Call `m.locate` (or `m.reset`). |
| `syncToTransport(m)` | Phase-lock a module added during playback. |
| `wireTransport(engine)` | Subscribe the engine's transport events to its modules' hooks. |
| `applyView()`, `s2w(x, y)` | Apply the pan/zoom transform; screen → scene coordinates. |
| `history.*` | `record, begin, end, run, mute, replace, undo, redo, clear, onChange`, plus getters `active/canUndo/canRedo/undoLabel/redoLabel`. |
| `recAddModule`, `recRemoveModule`, `recCable`, `recAmount`, `recParam`, `recMove`, `recCollapse` | Command factories (called after the action). |
| `editDepth(c, v, {merge})` | Set a cable depth and record it. |
| `placeModule(m, x, y)` | The single writer of module position. |
| `snapshotState(m)`, `stateEditor(m)` | Undo for free-form module state (e.g. SEQ patterns). |
| `userTempo(v)` | Undoable tempo change. |

### § 6 Modules & selection

| Function | Description |
|---|---|
| `addModule(type, x, y, id?)` | Build the module, attach header behaviour, register it, sync it to the transport, and record undo. |
| `measure(m)` | Cache the rendered size. |
| `removeModule(m)` / `removeModules(list)` | Full teardown (cables → snapshot → nodes → DOM) as one undo step. |
| `select(m, mode)` / `selectMany(list, mode)` / `pruneSelection()` / `syncSelection()` / `selectedMods()` | Selection model (`selSet`, `selMod`). |
| `beginModuleDrag(e, anchor, additive)` | Group move with a 3 px dead-zone; one undo step. |
| `boundsOf(mods)` | Bounding box in scene units. |

### § 7 Routing

| Function | Description |
|---|---|
| `mkTap(jack)` | Attach a 32-sample analyser so JS can read the jack's CV. |
| `canConnect(from, to)` | Type rules. |
| `connect(from, to, amount?)` | Create a connection (gain node, SVG, bookkeeping, undo). |
| `setAmount(c, a)` | Apply depth (scaled into parameter units for modulation). |
| `disconnect(c)` | Remove the connection (records undo first). |
| `rewireParam(p, targets)` | Move modulation cables to new AudioParams (used by LFO waveform changes). |
| `updPlugs()` | Light up connected plugs. |
| `jackEl(j)`, `jackPos(o)`, `wpath(a, b)`, `drawWire(c)`, `layoutWires()` | Jack DOM and cable geometry. |
| `selectConn(c)` | Select a cable. |
| `startCable(e, jack)` | Interactive cable drag, including picking up an existing cable from an input. |
| `fireTrig(jack, t, vel, dur, note)` / `fireRel(jack, t, note)` | Dispatch trigger events. |

### § 8–10 Inspector, navigation, clipboard, scheduler

| Function | Description |
|---|---|
| `showInsp()` | Rebuild the bottom bar (innerHTML with module titles). |
| `startMarquee(e)` | Rubber-band selection. |
| `fitView()` | Zoom to fit every module (0.28×–1.3×). |
| `isTextTarget(t)` | Is the event target an input/textarea/select/contentEditable? |
| `copyFragment()` / `pasteFragment(frag)` | Clipboard fragment build / place at the pointer. |
| `cloneBoundaryCables(mods, idMap, mode)` | Re-create cables crossing a duplicated selection. |
| `duplicateModules(mods, {cables})` | Duplicate to the right; one undo step. |
| `readTaps()` | Average each tapped jack's analyser buffer into `jack.cv`. |
| `pumpControl()` | `readTaps()` + apply all control-rate params. |
| `pumpSched(t, ahead)` | Call every module's `sched` (errors are swallowed). |

### § 11 Module definitions

`DEFS.osc`, `DEFS.poly`, `DEFS.drum`, `DEFS.filter`, `DEFS.crush`, `DEFS.echo`,
`DEFS.rev`, `DEFS.env`, `DEFS.lfo`, `DEFS.seq`, `DEFS.vca`, `DEFS.mix`, `DEFS.scope`,
`DEFS.out` and `DEFS.keys`; see § 5. Sequencer helpers:

| Function | Description |
|---|---|
| `seqDefaultSteps()` | 32 default steps (every 4th step on). |
| `seqPackSteps(S)` / `seqUnpackSteps(p)` | Columnar share codec for one pattern. |
| `m.isDefaultBank`, `m.fillBank`, `m.resetBank`, `m.setBank`, `m.refreshUI`, `m.stepDur`, `m.locate`, `m.recNote` | Per-instance SEQ API. |
| `m.noteOn(raw, vel)`, `m.noteOff(raw)`, `m.allOff()` | Per-instance KEYS API. |

### § 12–14 Palette, render loop, serialisation

| Function | Description |
|---|---|
| `frame(t)` | Main rAF loop. |
| `migrate(d)` | Validate and upgrade schema (`MIGRATIONS`). |
| `modById(id)` | Module lookup. |
| `exportModule(m)` | Module → JSON (`id, t, n, x, y, c, p, s, mc`). |
| `cableRef(c)`, `refKey(ref)`, `findConnByRef(ref)`, `connectRef(ref, lookup)` | Id-based cable references (survive module re-creation). |
| `exportFragment(mods, {cables})` | Modules (+ internal cables) → fragment. |
| `serialize()` | Whole rack + tempo + song → patch. |
| `importFragment(frag, {keepIds, dx, dy, at})` | Fragment → live modules/cables; returns `{modules, conns, idMap, skipped}`. |
| `restoreTitle(m, name)` | Restore a saved title and advance the per-type counter. |
| `clearPatch()` | Remove all modules. |
| `loadPatchData(d)` | Raw load (clear, song, tempo, import). |
| `deserialize(d)` | Undoable load plus fit view. |
| `loadBuiltin(name)` | Undoable built-in load. |
| `P_init`, `P_acid`, `P_drums`, `P_ambient` | Built-in racks. |

### § 15 Library & share

| Function | Description |
|---|---|
| `libRead()` / `libWrite(u)` | localStorage library I/O. |
| `patchBlob(name)` | `serialize()` + name + date. |
| `uniqueName(n, u)` | "foo" → "foo 2". |
| `dl(obj, filename)` | Download JSON. |
| `fdate(t)` | Short locale date. |
| `refreshPresets()` | Rebuild the toolbar dropdown. |
| `loadUser(name)` | Load a saved patch. |
| `renderLib()` | Rebuild the library dialog. |
| `saveAs(defName)` | Prompt and save. |
| `importObj(o, fname)` | Import a bundle or a single patch into the library. |
| `SHARE.makeUrl()` | Rack → URL. |
| `SHARE.decode(hash)` | Hash → patch object (size-capped). |
| `SHARE.copy(text)` | Clipboard with fallbacks (execCommand, then prompt). |
| `SHARE.report()` | Byte-cost table in the console. |
| internal: `b64.enc/dec`, `deflate`, `inflate` (capped), `pack(opt)`, `packSong(idx)`, `unpack(json)`, `sig4` | Codec pieces. |

### § 16–21

| Function | Description |
|---|---|
| `recStartFn()` / `recStopFn()` | Real-time recorder. |
| `encodeWav(L, R, len, sr)` | 16-bit WAV from Float32 blocks (recorder). |
| `setMidi(p, ch, cc)` | Attach a MIDI CC mapping. |
| `noteInput.*` | `listeners, on, off, panic, setQwerty, releaseQwerty, handleKeyDown, handleKeyUp, enableMidi, midiMsg, setLearn, armLearn, clearLearnTarget, cc`. |
| `song.*` | `bankFor(id, beat)`, `snapshot(prune)`, `restore(d)` (validated), `touch()`. |
| `applySongToTransport()` | Sync the loop region. |
| `songTick()` | Stop at the song end (song mode, no loop). |
| `songCommit(label, before)` / `songEdit(label, fn)` | Undoable song edits. |
| `songUI.toggle()` / `songUI.tick()` | Arranger panel (internal: `fits`, `trimToLength`, `syncHead`, `clipEl`, `renderGrid`). |
| `withHeadlessRack(engine, fn)` | Swap in a hidden rack bound to another engine, then restore. |
| `renderOffline({bars, tail, stems, onProgress})` | Offline bounce → `{files:[{name, L, R}], sampleRate}`. |
| `encodeWavPCM(L, R, sr, bits)` | 16/24-bit PCM or 32-bit float WAV. |
| `makeZip(files)` | Store-only ZIP writer (CRC-32). |
| `downloadBlob(blob, name)` | Trigger a download. |
| `exportUI.open()` | Bounce dialog (internal `go()`, `close()`). |
| `boot(which)` | Engine init → load patch → autoplay. |

---

## 14. Known issues

These come from a full read of the source. Items marked **✔ verified** were reproduced
in headless Chromium 141.

### Summary

| # | Severity | Area | Issue |
|---|---|---|---|
| C1 | Critical | Engine | Noise buffers are empty: OSC white/pink, every DRUM noise layer and the whole **clap** engine are silent ✔ |
| C2 | Critical | OSC | `Trig` input calls undefined `fire`/`rel`. Patching a gate into it **freezes the sequencer**, which then machine-guns its current step ✔ |
| C3 | Critical | ENV | No `onRel` handler: notes from KEYS/MIDI never release, so the envelope sticks at sustain ✔ |
| C4 | Critical | Share | Share links lose **all sequencer patterns** (version 1 written, version 2 codec used) ✔ |
| H1 | High | Security | HTML/script injection via patch names and module titles in imported files ✔ |
| H2 | High | Share/Load | A malformed or hostile link (e.g. module type `constructor`) wipes the current rack **without undo** ✔ |
| H3 | High | KEYS | `m.def.free` is shared by all instances: deleting one KEYS releases another's notes ✔ |
| H4 | High | OSC | **FM Amt** knob is dead; nothing feeds the FM gain node ✔ |
| M1 | Medium | Toolbar | Duplicate `#clr` handler: **clear** is not one undo step (13 steps for the acid patch) and skips the song reset ✔ |
| M2 | Medium | REVERB | Modulating Size/Decay/Damp/Diffuse does nothing ✔ |
| M3 | Medium | Params | Pitch knobs with `cents:1` display wrong modulated values ("0¢" while at +1200¢) ✔ |
| M4 | Medium | ENV | Loop mode never plays the release stage |
| M5 | Medium | LFO | Tempo-synced periodic LFOs ignore BPM changes and aren't phase-locked; LFO oscillators leak on delete ✔ |
| M6 | Medium | ECHO | Feedback range goes to 1.15, causing unbounded growth; 🎲 can land there |
| M7 | Medium | Output | Master "limiter" is a compressor: bounces peak above 0 dBFS (1.165 observed) and clip in 16/24-bit ✔ |
| M8 | Medium | Bounce | Ignores the master slider; 80 ms offset at the start ✔; silent 600 s cap; stems are pre-limiter/pre-master |
| M9 | Medium | Song | Loading a built-in or clearing keeps the old song state (song mode/loop/length) |
| M10 | Medium | Timing | Control-rate modulation and the SEQ playhead queue live in `requestAnimationFrame` |
| L1–L12 | Low | various | Polish, dead code, docs drift (see below) |

---

### C1 · Noise buffers are never filled

`mkNoise()` (≈ line 817) contains a leftover placeholder from a refactor:

```js
function mkNoise(ac, pink) {
  const len = ac.sampleRate * 3;
  const buf = ac.createBuffer(1, len, ac.sampleRate);
  /* … rest of the original mkNoise body, unchanged … */
  return buf;            // ← all zeros
}
```

**Effect:** OSC waves *white* and *pink* are silent. The noise parts of kick, snare,
hat, clap, tom, rim and zap are missing, and **clap makes no sound at all**.
Verified: both buffers have a peak of 0.

**Fix:** restore the generator, e.g. white = `Math.random()*2-1`; pink = the Paul
Kellet filter on white noise, normalised.

### C2 · OSC `Trig` input references undefined functions

Line ≈ 2785: `onTrig: (t, v, d) => fire(t, v, d), onRel: t => rel(t)`. Neither `fire`
nor `rel` exists inside `DEFS.osc.build`; the `onRel` comment ("held gate (KEYS) ends
here") suggests it was pasted into the wrong module (compare C3).

**Effect:** a `ReferenceError` is thrown inside the source sequencer's `fire()`. It
happens *before* `st.nextT` advances, and `pumpSched` swallows the error. As a result,
the sequencer:

* freezes on one step,
* re-fires it on every 22 ms tick (42 retriggers per second observed on an ENV patched
  to the same gate),
* no longer delivers triggers to receivers listed after the OSC.

**Fix:** remove the OSC Trig jack, or implement it (e.g. phase reset / hard sync).
Also isolate per-receiver errors in `fireTrig` with a `try/catch`, so one faulty
receiver can't stall a sequencer.

### C3 · ENV ignores note-off

`DEFS.env` registers only `onTrig` (line ≈ 3863). KEYS sends held events
(`dur = null`) and later calls `fireRel`, but ENV has no `onRel`. Verified: after
note-off the envelope stays at the sustain level indefinitely. This makes the
KEYS/MIDI → ENV → VCA workflow unusable.

**Fix:** `onRel: t => rel(t)` on the ENV Trig jack.

### C4 · Share links drop sequencer state

`SHARE.pack()` writes `v: VERSION` with `VERSION = 1` (line ≈ 5892), but stores SEQ
state through `def.pack()` (the compact v2 `{k, a}` form). `unpack()` only calls
`def.unpack` when `o.v >= 2` (line ≈ 5994). The compact object is therefore handed to
`DEFS.seq.load`, which finds neither `banks` nor `steps` and keeps the defaults.
Verified with the acid patch: the pattern returns as the default four-on-the-floor,
and song clips keep pointing at empty patterns.

**Fix:** `const VERSION = 2;` (the decoder already accepts 1 and 2; old v1 links keep
working).

### H1 · HTML injection from patch files

`h()` and `toast()` use `innerHTML`, and several call sites interpolate user- or
file-controlled strings:

* the library row name (`h('div','lname',k)`, ≈ line 5698);
* `toast('loaded "' + name + '"')`, `toast('imported "'+nm+'"')`, `toast('bad file: '+f.name)`;
* `showInsp()` (≈ lines 2412/2460), which inserts module titles; titles come from
  the `n` field of imported patches via `restoreTitle`.

A crafted `.json` with `"name": "<img src=x onerror=…>"` or a module `"n"` like that
runs script as soon as the library opens, and again when the module is selected.
Verified: an injected `onerror` handler fired 6 times. Share links are *not* affected
today, because `unpack` drops `n` and names. Library bundles shared between users
are.

**Fix:** use `textContent` for data (give `h()` a text variant, or escape it). Also
validate `n` (e.g. require `^[A-Z]+ \d+$`) or drop it on import.

### H2 · A bad share link destroys the current rack

`DEFS` is a plain object, so `DEFS['constructor']`, `DEFS['toString']` etc. are
truthy. `importFragment` accepts such a type, and `addModule` then throws
`def.build is not a function`. Because `loadPatchData` has already called
`clearPatch()`, and `history.replace` only records after `fn()` returns, the user is
left with a partially loaded rack and **no undo entry**. Verified: a 10-module rack
became 1 module, with `canUndo === false`.

**Fix:**

* Use `Object.hasOwn(DEFS, t)` (or `Object.create(null)`) everywhere a type is looked up.
* Build the new rack before clearing, or make `history.replace` record the "before"
  snapshot in a `finally`.
* Wrap each module build in a `try/catch` during import.

### H3 · KEYS overwrites the shared definition's `free`

Line ≈ 5025: `m.def.free = () => m.allOff();` assigns onto `DEFS.keys`, so the *last
built* KEYS instance wins. Offline renders build a hidden KEYS too. Verified: deleting
KEYS #1 called `allOff()` on KEYS #2, leaving #1's held notes stuck in connected
POLY/ENV modules.

**Fix:** use a per-instance hook (e.g. `m.free`) and have `removeModule` call it. See
M5 for the matching LFO bug.

### H4 · OSC "FM Amt" has no input

`const fm = AC.createGain(); fm.connect(o.frequency)` (≈ line 2771), but no jack or
node ever connects *into* `fm`, so the knob scales silence. The module description
promises "sub & FM". (Exponential FM still works by patching audio onto the **Pitch**
knob.)

**Fix:** `m.addIn({ id:'fm', label:'FM', kind:'sig', node: fm })`.

### M1 · "clear" is registered twice

Line ≈ 5848 sets an undoable `history.replace('clear rack', …)` handler, and line
≈ 6199 overwrites it with `clearPatch(); P_init();`. Verified: one clear created
**13 separate undo steps**. Neither version resets the song (see M9).

**Fix:** delete the second assignment.

### M2 · Reverb shape modulation is ineffective

Size/Decay/Damp/Diffuse are control-rate parameters whose `set` is `queueIR`, a 140 ms
debounced rebuild.

* When modulated, `set` is called every frame, so the debounce timer restarts every
  frame and the IR is **never** rebuilt while a cable is attached.
* `buildIR()` reads `m.params.*.value` (the base value), not `effective()`, so even a
  rebuild would ignore the modulation.

Verified: the IR length stayed at 97,020 samples while the modulated size read 3.97 s.
The knobs still show yellow rings, which is misleading.

**Fix:** either remove `mod` from these knobs, or use `effective()` with a throttled
(not debounced) rebuild on significant change.

### M3 · Wrong modulated readout for linear-cents knobs

`P.map()` treats `cents:1` as multiplicative: `base · 2^(cv·scale/1200)`. That's right
for Hz knobs (cutoff, tone), but wrong for OSC/POLY **Pitch**, whose base value is
itself in cents (0 by default). With a base of 0, `effective()` is always 0. Verified:
the OSC readout showed "0¢" while the real offset was +1200 ¢, and the modulation ring
collapses to a dot. Audio is correct, because the cable goes straight to `detune`.

**Fix:** drop `cents:1` from the two Pitch knobs (linear `base + cv·scale` is correct
there).

### M4 · ENV Loop skips the release

In loop mode `m.sched` calls `fire(t, 1, dly+a+hold+d)`, but `fire` only schedules
the release when `!q.loop`. Each cycle therefore sits at sustain for the "release"
portion of the period, then jumps to 0, which clicks. The visual suggests a full
DAHDSR loop.

**Fix:** schedule the release whenever `dur != null`.

### M5 · LFO tempo sync & teardown

* The LFO defines no `m.setT`, so a synced periodic LFO keeps its old rate after a BPM
  change, until the Wave or Sync select is touched. It also has no `locate`, so synced
  LFOs are not phase-aligned to the bar.
* The oscillator created by `remake()` is not registered with `m.node()`. `m.free`
  (≈ line 4157) is never called, because `removeModule` only calls `m.def.free`.
  Deleted LFOs therefore leave a running `OscillatorNode` behind, and so does every
  retrigger.

**Fix:**

* Add `m.setT = () => osc && setAP(osc.frequency, rateHz(), .02)`, plus a `locate`
  that calls `remake(time)` with a phase offset.
* Call a per-instance `m.free` in `removeModule`.

### M6 · Echo feedback above unity

`Fb` goes up to 1.15, and modulation (scale 0.5) can push it further. The loop only
contains biquads (no saturation), so the signal grows exponentially. The only
protection is the soft master compressor (M7). The 🎲 button can pick these values.

**Fix:** cap the feedback at ≤ 0.98 or add a `tanh` shaper inside the loop; mark `fb`
as `nornd` or limit its random range.

### M7 · No true brick-wall limiting

Both the per-OUT "Limiter" and the master chain are `DynamicsCompressorNode`s
(threshold −6/−4 dB, ratio 12–14). They let peaks through. Verified: an acid-patch
bounce peaked at **1.165** (+1.3 dBFS), which clips hard in the 16/24-bit exports.

**Fix:** add a final `WaveShaper` soft-clipper, or apply peak normalisation/clamping
with a warning at export.

### M8 · Bounce fidelity

* **Level differs from live.** The offline engine uses the default master gain (0.72),
  not the toolbar **master** slider, so bounces are louder or quieter than what you
  heard.
* **Late start.** `transport.play()` starts 80 ms late (`START_DELAY`), so every bounce
  begins with ~80 ms of silence and is 80 ms short at the end. Verified: first sound at
  92 ms.
* **Silent truncation.** The render is silently capped at 600 s
  (`RENDER_LIMIT_SEC`), and the UI doesn't say so.
* **Stems don't sum to the master.** Stems are tapped pre-OUT-limiter and pre-master
  gain/compressor. This is fine for stems, but it isn't documented in the dialog.

### M9 · Song state leaks across patch loads

`loadBuiltin()` and both **clear** handlers never call `song.restore()`. After you've
used song mode (no loop) on another patch, loading a built-in keeps song mode on.
Playback then stops after the old `bars` length with "song finished", even though the
new patch has no clips.

**Fix:** call `song.restore()` in `clearPatch()` or `loadBuiltin()`.

### M10 · Work tied to `requestAnimationFrame`

* Control-rate modulation (`pumpControl`) runs only in the rAF loop. rAF stops in
  background tabs and minimised windows, so modulation of drum knobs, SEQ
  length/swing/transpose, echo time and similar freezes while you're in another tab.
  (The 22 ms scheduler keeps running; audible tabs are generally exempt from timer
  throttling, but that is browser-dependent.)
* `SEQ.st.q` (the playhead queue) is drained only in `m.draw`, which runs only for
  on-screen, expanded modules. It grows without bound while a SEQ is scrolled off
  screen or collapsed, and song-driven pattern view changes are missed.

**Fix:** run `pumpControl` from the scheduler interval too, and drain `st.q` in
`sched`.

### Low-severity items

| # | Issue |
|---|---|
| L1 | **Debug logging left in:** `console.log(ORDER)`, `console.log(DEFS)` and a log per palette type (≈ lines 5093–5098). |
| L2 | **Malformed help HTML:** the "Edit:" `<li>` is never closed; the "Play"/"Song" items nest inside it; the orphan fragment "(Shift+click ⧉ does the same).</li>" renders after the Song item (≈ lines 617–633). |
| L3 | **Dead or duplicated code:** unused `setPlay()`; an empty "AudioParam utilities" doc block (≈ line 902); `history.clear()` called twice in `boot`; `posEl.textContent` written twice per frame; `encodeWav` (recorder) duplicates `encodeWavPCM`; per-module boilerplate destructures unused `cah/BUS/WT/NZ_*`; a fresh `Float32Array(256)` is allocated every frame for the meter. |
| L4 | **Docs drift:** the SECTION INDEX lists 1–18 only (19–21 exist); it says the Recorder is "16-bit"/"master tap" while the bounce supports more; the module-object comment lists `m.reset` but not `m.locate`; `SCHEMA` comments mention "Patch S3/Step 6" leftovers. |
| L5 | **Scale quantiser isn't octave-wrap aware:** `quantN` never considers the next octave's root, so in PentMaj a pitch class of 11 snaps down to 9 (distance 2) instead of up to 12 (distance 1). |
| L6 | **Swing with odd step counts:** swing uses step-index parity, so with odd `Steps` the long/short feel flips every loop. |
| L7 | **Acid preset drone:** SEQ Vel → VCA Gain at depth 0.9 keeps the VCA at ≥ 0.63 between notes (verified min 0.63), so the oscillator never fully closes; the ENV → VCA cable adds only the accent. (Probably intended to be much lower, or Vel → ENV.) |
| L8 | **Echo Freeze:** turning the Fb knob while frozen overrides the frozen unity feedback. |
| L9 | **Recording source mismatch:** the SEQ **● rec** tooltip promises recording from "KEYS", but notes played on a KEYS module's on-screen keyboard bypass `noteInput` and are not recorded. |
| L10 | **Real-time recorder:** "bars" counts from the click, not from a bar line; it uses the deprecated `ScriptProcessorNode` on the main thread (UI jank can cause dropouts). |
| L11 | **No feedback-cycle handling:** Web Audio mutes cycles that contain no `DelayNode`, so patching e.g. FILTER Out → FILTER In silently kills that path without any UI hint. |
| L12 | **UI details:** the shortcut handler runs while overlays are open (Space plays behind the library). Esc does not close the bounce dialog or the song panel. Changing Shift mid-drag makes knobs jump (fine mode is computed from the drag origin). Canvas resolution is fixed at the `devicePixelRatio` of creation time. Worklet blocked on `file://` in Chromium (see § 1). The AudioContext is constructed before any user gesture, which logs a warning. `localStorage` names such as `__proto__` misbehave in the library object. |

### Suggested priority

1. C1–C4 and H4: small, local fixes that restore advertised features.
2. H1/H2: escape all data going into `innerHTML`, harden type lookups, and make loads
   atomic.
3. H3, M1, M2, M3, M4, M5: correctness of individual modules and the toolbar.
4. M6–M10: output safety, bounce fidelity and background behaviour.
5. A tiny regression harness: the app already runs fine in headless Chromium (via
   Playwright), and most issues above were checked that way (load each preset, bounce
   two bars, assert peak/silence, round-trip a share link).
