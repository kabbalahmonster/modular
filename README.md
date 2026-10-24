# ◆ MODULAR — a serverless web modular synthesizer & mini-DAW

MODULAR is a patchable synthesizer, drum machine, step sequencer, song arranger and
offline renderer that lives in **one static HTML file** (`index.html`, ~6,900 lines,
no dependencies, no build step, no network calls).

You drag cables from outputs to inputs (or onto knobs, to modulate them), sequence
patterns, arrange them into a song, and bounce the result to WAV. Patches are saved
in the browser, exported as JSON, or **shared as a self-contained URL**: the whole rack
is compressed into the link's `#hash`, so no server is ever involved.

> **Status note:** a code review found 4 critical, 4 high, 10 medium and 12 low
> issues. All 30 are fixed in this revision and were re-verified in headless
> Chromium. See [§ 14 Changelog & known issues](#14-changelog--known-issues).

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
14. [Changelog & known issues](#14-changelog--known-issues)

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

**`file://` and `http(s)://` both work.** The AudioWorklets (CRUSH DSP and the
recorder tap) load from a Blob URL; where that is refused (Chromium on `file://`) they
fall back to a `data:` URL. Only browsers without AudioWorklet use the
`ScriptProcessorNode` fallback; the start-up toast then reads *"ready · fallback dsp"*
instead of *"ready · worklet dsp online"*.

No AudioContext exists until you click a splash button, so browsers don't log
autoplay warnings.

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
* Feedback is allowed. A cable that closes an audio loop is automatically routed
  through a one-render-quantum delay (≈ 3 ms), which the Web Audio spec requires for
  cycles. Without it, Firefox/Safari would mute the loop. The inspector marks such
  cables "feedback (+3 ms)".

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
| **bars** | 0 = record until you press ● again; N = exactly N bars, starting on the next bar line (§ 10). |
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
| Knob | drag up/down | Change the value (**Shift** = fine ×¼; you can press or release Shift mid-drag without a jump). One undo step per drag. |
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
| **Esc** | Close library/help/bounce dialogs and the song panel, disable piano & MIDI learn, leave a text field, deselect (works even while typing) |

Keys are ignored while the focus is in a text field, during an offline render, and
while a dialog (splash, help, library, bounce) is open; **Esc** always works.

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

A free-running oscillator with linear FM input, sub-oscillator, pulse-width shaping,
two noise colours, four additive wavetables and slow analogue-style drift.

* **Inputs:** `FM` (sig). Any signal patched here is multiplied by **FM Amt** (in Hz)
  and added to the oscillator frequency: linear, through-zero-capable FM. Patch
  another OSC here for classic FM tones. For vibrato or exponential FM, patch onto the
  **Pitch** knob instead.
* **Outputs:** `Out` (sig)

Signal flow: `FM in → FM Amt → osc.frequency`; `osc → (wave crossfade) ┬ sub ┬ white
┬ pink → Level → Out`. *Pulse* is a sawtooth plus a DC offset pushed through a hard
`tanh` shaper, so moving the offset (PW) moves the duty cycle.

> Older versions had a non-functional `Trig` input here. Patches that used it load
> without that cable.

| Knob | Range | Default | Curve | Mod | Scale | Notes |
|---|---|---|---|---|---|---|
| Oct | −4…+4 | 0 | lin, step 1 | ctrl | 4 | Octave of the base pitch. Not randomised. |
| Semi | −12…+12 | 0 | lin, step 1 | ctrl | 12 | Semitone offset. Not randomised. |
| Pitch | −1200…+1200 ¢ | 0 | lin | audio → `detune` | 4800 ¢ | **Main pitch-CV destination.** Also drives the sub. |
| PW | −0.92…+0.92 | 0 | lin | audio | 0.9 | Pulse width; only audible with *pulse*. |
| Sub | 0…1 | 0 | pow2 | audio | 1 | Square sub-oscillator level. |
| SubOct | 1…2 | 1 | lin, step 1 | – | – | Sub sits 1 or 2 octaves down. |
| FM Amt | 0…4000 Hz | 0 | pow3 | audio | 2000 | Depth of the `FM` input, in Hz of deviation per unit of signal. |
| Drift | 0…1 | 0.08 | lin | – | – | Random detune (up to ±34 ¢), re-targeted every scheduler tick. |
| Level | 0…1.2 | 0.5 | pow2 | audio | 1 | Output level. |

**Wave:** `sine, triangle, saw, square, pulse, organ, glass, vocal, digi, white, pink`.
`organ/glass/vocal/digi` are `PeriodicWave`s defined in `initTables()`. `white/pink`
are looping 3-second noise buffers: uniform white noise, and pink noise from Paul
Kellet's filter. Both are normalised to a 0.95 peak.

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

The **trigger** pad fires a manual hit.

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

Two 4-second delay lines with cross-feedback through HP → LP filters and a soft
saturator, panned hard L/R. An internal LFO ("wobble") modulates both delay times for
tape flutter/chorus.

The saturator is unity gain for normal levels and limits the loop to about ±1.5. With
**Fb** above 1.0 the echo therefore self-oscillates at a bounded level, like a tape
echo driven into runaway.

| Control | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Sync | free, 16/1 … 1/32 (incl. triplets) | 1/8 | – | Tempo-synced delay time. Capped at 3.9 s. |
| Time | 5 ms…2 s (exp) | 0.3 s | ctrl, 0.5 | Only used when Sync = free. |
| R ratio | 0.25…2 | 1 | ctrl, 0.75 | Right delay = left × ratio. |
| Fb | 0…1.15 | 0.45 | audio, 0.5 | Feedback. Above 1.0 = bounded self-oscillation. 🎲 only picks 0…0.85. |
| Tone | 300 Hz…18 kHz | 5200 | audio → `detune`, 3600 ¢ | Feedback low-pass. |
| HP | 20 Hz…2 kHz | 120 | audio → `detune`, 2400 ¢ | Feedback high-pass. |
| Wobble | 0…1 | 0 | audio, 0.01 | Flutter depth (up to 4 ms). |
| W.Rate | 0.05…8 Hz | 0.4 | audio, 4 | Flutter rate. |
| Mix | 0…1 | 0.35 | audio (wet), 0.5 | Dry is reduced to `1 − 0.35·mix`. |
| Freeze (toggle) | – | off | – | Closes the input and sets feedback to 1.0: an infinite loop. While frozen, turning Fb has no effect until you unfreeze. |

### 5.8 REVERB — generated convolution reverb (`rev`)

A `ConvolverNode` whose stereo impulse response is synthesised in JS: filtered noise,
a power-law decay and sparse early reflections, normalised.

Changes to Size, Decay, Damp and Diffuse (including modulation) rebuild the IR. The
rebuild is **throttled** to at most one per 140 ms and skipped when the change is
inaudibly small. Modulating these knobs therefore gives stepped, grainy changes to the
room (each rebuild also re-rolls the noise), not smooth sweeps. For smooth motion,
modulate PreDly, LowCut, HiCut or Mix.

| Knob | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Size | 0.15…8 s | 2.2 | ctrl (stepped) | IR length. |
| Decay | 0…1 | 0.4 | ctrl (stepped) | Decay curve exponent. |
| Damp | 0…1 | 0.45 | ctrl (stepped) | High-frequency damping over time. |
| Diffuse | 0…1 | 0.3 | ctrl (stepped) | Early-reflection strength. |
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

* **Inputs:** `Trig` (trig). A timed event (`dur` given, e.g. SEQ) releases after `dur`. A held event (`dur = null`, from KEYS, MIDI or the piano) sustains until the note-off.
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
* **Loop** re-fires the envelope free-running with period `dly+a+hold+d+r`. Every
  cycle plays the full shape, including the release.
* The **gate** pad behaves like a held key: press = attack, release = release.

### 5.10 LFO — low-frequency oscillator / random source (`lfo`)

* **Inputs:** `Retrig` (trig). Restarts the periodic oscillator's phase. Random modes ignore it.
* **Outputs:** `LFO` (sig), always tapped.

| Control | Range | Default | Mod | Notes |
|---|---|---|---|---|
| Wave | sine, triangle, saw, square, s&h, random, drift | sine | – | First four use an `OscillatorNode`. `s&h` = stepped random values; `random` = linear ramps between them; `drift` = smooth exponential glide. |
| Sync | free, 16/1 … 1/32 | free | – | Rate from the tempo (cycle = division length). Synced LFOs follow BPM changes and are **phase-locked to the transport**: on play/seek/loop the oscillator restarts at the phase the song position implies, so a 1/1 LFO always begins its cycle on the downbeat. |
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
| Limiter | toggle | on | – (compressor −6 dB/14:1, then a brick-wall soft clipper at ±0.99) |

`Thru` outputs the post-pan, **pre-limiter** signal (handy for a SCOPE). The visual is
a peak meter with a clip flag.

**Master bus** (global): `bus gain (master slider, 0.72 default) → 12 Hz DC-blocking
high-pass → compressor (−4 dB, 12:1, 3 ms attack) → soft clipper → analyser/meter →
speakers`.

The soft clipper (`mkSoftClip`) is linear below 0.8 and has a smooth `tanh` knee
above that, so the output **never exceeds ±0.99**: live, recorded and bounced audio
cannot clip. The compressors add a few ms of lookahead latency (about 6 ms each).

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
| Swing | 0…0.7 | 0 | ctrl | Even steps of the *beat grid* are lengthened by (1+sw), odd ones shortened by (1−sw). Because it follows the grid (not the pattern index), the feel stays put with odd lengths and non-forward directions. |
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

With **● rec** armed, notes from the **QWERTY piano, Web MIDI or a KEYS module's
on-screen keyboard** are written into the current pattern:

* **Stopped = step record.** Each note fills the next step and advances the cursor.
* **Playing = live record.** The note snaps to the nearest step (a note played more
  than half a step late, in forward mode, goes to the next step).

Notes played on a KEYS module's *on-screen* keyboard are recorded too. Incoming notes
are converted back to scale degrees relative to Root/Transp/Oct and folded into the
0…24 range.

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
string, and other lanes only when they differ from the defaults.

* Because the fragment never reaches a server, the patch is never uploaded anywhere.
* On first load, a link adds **⇩ Open shared patch** to the splash. Pasting a link
  into an already-open tab loads it immediately as an undoable step.
* Links are treated as hostile input:
  * safety limits of 2 MB decompressed, 200 modules and 2000 cables;
  * unknown module types (including prototype names like `constructor`) are skipped;
  * values are validated;
  * a load that fails part-way is rolled back, so your current rack is never lost.
* Links longer than 8000 characters trigger a warning, because some apps truncate them.
* In the console, `await SHARE.report()` shows what each part of the patch costs in bytes.

### Clipboard fragments

Copy/cut puts a JSON **fragment** (the same schema, `kind: "fragment"`, with internal
cables only) on the system clipboard, so you can paste between tabs or save snippets
in a text file.

---

## 10. Recording & bouncing audio

### ● Real-time recorder

* Taps the post-limiter master (what you hear) and writes a 16-bit stereo WAV named
  `modular-<timestamp>.wav`.
* **bars = 0:** records from the click until you press ● again.
* **bars = N:** records **exactly N bars**, starting on the **next bar line**:
  * If the transport is stopped, ● starts it, and the take begins with the transport.
  * The end is defined in beats, so tempo changes during the take are honoured.
  * Stopping the transport early ends the take there.
  * A toast says which bar the take starts on.
* Capture runs on the audio thread in the `rec-tap` AudioWorklet, so UI activity
  can't cause dropouts. Every block is time-stamped and the file is cropped
  sample-exactly (e.g. 2 bars at 120 BPM = 4.000 s). Browsers without AudioWorklet
  use a ScriptProcessor fallback with the same cropping.

### Bounce (offline render)

**bounce** opens a dialog:

| Field | Meaning |
|---|---|
| length (bars) | Defaults to the song length. 1–512 bars; the total is capped at 10 minutes (the dialog reports when this happens). |
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

Further details:

* If song mode is on, the arrangement is used; looping is disabled offline.
* The render starts at exactly t = 0: there is no 80 ms live lead-in, so the first
  downbeat lands at the start of the file (after the compressors' few-ms lookahead).
* The master file uses the current **master** slider level, and goes through the same
  soft clipper as the live output.
* Renders are limited to **10 minutes**. If the requested length is longer, the
  dialog and a toast say so ("capped at 10:00").
* Stems are taken from each OUT module *after* its Level/Pan, but *before* its limiter
  and the master bus. They are raw material for mixing, so they will not sum exactly
  to the master.
* The ZIP writer is a built-in store-only implementation.

---

## 11. Built-in patches

| Name | Contents |
|---|---|
| **init / Empty Rack** | OUT → SCOPE. |
| **acid** (128 BPM) | SEQ (PentMin, swing, slides, a ratchet) → OSC saw + sub → FILTER 24 dB (resonant; modulated by ENV + a slow LFO) → CRUSH → VCA (opened by the ENV, whose velocity sensitivity carries the accents, so it closes fully between notes) → ECHO 1/8T → OUT → SCOPE. |
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
| 2 | Audio engine core | `createEngine` (detached or attached; master chain, noise, wavetables, worklets), `ENGINE` singleton, `startAudio`, worklet source |
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

`createEngine(ac)` wraps an `AudioContext` *or* an `OfflineAudioContext`. Pass `null`
to create it detached: the transport exists immediately, and `attach(ac)` later builds
the master chain. The live app does exactly that. `ENGINE = createEngine(null)` at
load, and `startAudio()` (called synchronously from the splash click in `boot()`)
creates the AudioContext inside the user gesture. Offline renders pass their context
directly. The engine exposes:

* `ac` (null until attached), `attach(ac)`, `now()` (0 while detached)
* `setAP(param, value, tc)`: a click-free `setTargetAtTime`.
* `cah(param, t)`: cancel-and-hold.
* `bus`/`masterGain`, `masterAna`
* `noise {white, pink}`, `waves` (PeriodicWaves), `hasWorklet`
* `transport`
* `init({workletSrc})`: async; creates the noise buffers and wavetables, and registers
  the worklets (`crusher` and `rec-tap`) from a Blob URL, falling back to a `data:` URL.

App-shell code (toolbar, recorder, scheduler) uses the thin wrappers `now()`,
`setAP()`, `transport` and the `AC` variable (null before `startAudio()`).

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
* `m.viz(height)` → a HiDPI 2D context (`ctx.canvas.W/H` in CSS pixels), registered in
  `m.vizList` so it is re-rasterised when `devicePixelRatio` changes
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
| `m.setT()` | tempo change | recompute tempo-synced times (ECHO, LFO) |
| `m.free()` | `removeModule` | per-instance teardown (LFO oscillator, KEYS held notes, REVERB timer) |
| `m.onDpr(ratio)` | `devicePixelRatio` changed | resize extra canvases not created with `m.viz` (SCOPE's trail buffer) |
| `m.trigNow(t, vel, dur)` | – | manual trigger convenience |

Optional `def` hooks:

| Hook | Purpose |
|---|---|
| `save(m)` / `load(m, s)` | persistent state (patch files, clipboard) |
| `snap(m)` | undo-relevant subset of the state (falls back to `save`) |
| `pack(m)` / `unpack(p)` | compact share-link codec |
| `free(m)` | per-*type* teardown. Never assign `m.def.free` from inside `build()`: `def` is shared by every instance. Use `m.free` instead. |

### Parameter definitions

```js
{ k: 'knob'|'select'|'toggle', id, label,
  min, max, def, step?, curve: 'lin'|'exp'|'pow2'|'pow3', unit?, fmt?,
  opts?,                    // selects
  cents?: true,             // modulation is exponential (base · 2^(cv·scale/1200))
  nornd?: true,             // excluded from 🎲
  rnd?: [lo, hi],           // safe range for 🎲, in parameter units (e.g. echo feedback)
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
* Receivers: ENV (hold/release), POLY (voice allocation), DRUM (ignores note and
  duration), LFO Retrig, SEQ Reset.
* Helpers: `fireTrig(jack, …)` and `fireRel(jack, …)` walk `jack.conns`. An exception
  in one receiver is logged and skipped, so it can never stall the sender (e.g. freeze
  a sequencer).

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

* **Audio events and control-rate modulation:** a `setInterval(22 ms)` loop runs
  `pumpControl()` (read taps, apply control-rate params), calls `m.sched(t, t+0.12)`
  on every module, and handles seamless loop wrapping (`loopCrossing`/`wrapLoop`).
  Because this loop is a timer, modulation keeps working in background tabs, where
  rAF is paused.
* **Visuals:** the `requestAnimationFrame` loop (capped at ~66 fps) also refreshes the
  taps for smooth knob rings, draws only on-screen modules, re-lays cables when
  `wiresDirty`, and updates the meter, position readout and song panel.

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
5. Never put patch/file data into markup: use `ht()` (text) or `esc()`; `h()`'s third
   argument is parsed as HTML.

---

## 13. Function reference

Line numbers are approximate (they refer to the current `index.html`).

### § 1 Helpers

| Function | Description |
|---|---|
| `h(tag, cls, html)` | Create an element; `html` is assigned via **innerHTML**. Only for markup written in this file. |
| `ht(tag, cls, text)` | Create an element with plain **text** content (safe for names from files/links). |
| `esc(v)` | HTML-escape a value for template strings. |
| `$(sel)` | `document.querySelector`. |
| `clamp(v, a, b)`, `lerp(a, b, t)` | Maths. |
| `rnd(max=1, min=0)` | Random float in [min, max). Note the reversed argument order. |
| `toast(text)` | Transient status message (plain text). |

### § 2 Engine

| Function | Description |
|---|---|
| `createEngine(ac)` | Build an engine around a (possibly offline) context: master chain, `setAP`, `cah`, transport and async `init`. |
| `mkSoftClip(ac)` | Brick-wall soft clipper (`{input, output}`): linear below 0.8, `tanh` knee, ceiling ±0.99, 2× oversampled. Used on the master bus and in each OUT limiter. |
| `engine.attach(ac)` | Bind a context and build the master chain (idempotent). |
| `engine.init({workletSrc})` | Resume the context; create noise buffers and wavetables; register the worklets (Blob URL, then `data:` URL). |
| `startAudio()` | Create the live AudioContext on the first user gesture, attach it, apply the master slider. |
| `mkNoise(ac, pink)` | Create a 3 s mono noise buffer: white (uniform) or pink (Paul Kellet filter), normalised to a 0.95 peak. |
| `mkWave(ac, table, name, harmonics)` | Build a `PeriodicWave` from a cosine harmonic list. |
| `initTables(ac, table)` | Define the `organ`, `glass`, `vocal` and `digi` wavetables. |
| `setPlay(p)` | Legacy play/stop wrapper (unused). |
| `clock` | Legacy facade: `clock.bpm` get/set, `clock.playing`. |
| `WORKLET_SRC` | AudioWorklet source: `crusher` (bit crusher DSP) and `rec-tap` (recorder capture, 4096-frame time-stamped chunks). |

### § 3 Music & transport

| Function | Description |
|---|---|
| `noteName(m)` | MIDI number → `"C4"`. |
| `mtof(m)` | MIDI number → Hz (A4 = 440). |
| `quantN(n, scale, root)` | Snap semitone `n` to the nearest scale degree relative to `root`, considering the neighbouring octaves (ties resolve downward). |
| `createTransport(engine)` | Beat clock with `on/emit`, `beatSec`, `beatAt`, `timeAt`, `setTempo`, `play(fromBeat?, delay = 0.08)`, `stop`, `seek`, `setLoop`, `loopCrossing` and `wrapLoop`. Live playback starts 80 ms ahead for clean scheduling; offline renders pass `delay = 0`. |
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
| `history.*` | `record, begin, end, run, mute, replace, undo, redo, clear, onChange`, plus getters `active/canUndo/canRedo/undoLabel/redoLabel`. `replace()` is atomic: if the change throws, the previous rack is restored before the error propagates. |
| `recAddModule`, `recRemoveModule`, `recCable`, `recAmount`, `recParam`, `recMove`, `recCollapse` | Command factories (called after the action). |
| `editDepth(c, v, {merge})` | Set a cable depth and record it. |
| `placeModule(m, x, y)` | The single writer of module position. |
| `snapshotState(m)`, `stateEditor(m)` | Undo for free-form module state (e.g. SEQ patterns). |
| `userTempo(v)` | Undoable tempo change. |

### § 6 Modules & selection

| Function | Description |
|---|---|
| `addModule(type, x, y, id?)` | Build the module, attach header behaviour, register it, sync it to the transport, and record undo. If `build()` throws, the partial module is torn down and `null` is returned. |
| `measure(m)` | Cache the rendered size. |
| `removeModule(m)` / `removeModules(list)` | Full teardown (cables → snapshot → `m.free()` / `def.free(m)` → nodes → DOM) as one undo step. |
| `select(m, mode)` / `selectMany(list, mode)` / `pruneSelection()` / `syncSelection()` / `selectedMods()` | Selection model (`selSet`, `selMod`). |
| `beginModuleDrag(e, anchor, additive)` | Group move with a 3 px dead-zone; one undo step. |
| `boundsOf(mods)` | Bounding box in scene units. |

### § 7 Routing

| Function | Description |
|---|---|
| `mkTap(jack)` | Attach a 32-sample analyser so JS can read the jack's CV. |
| `canConnect(from, to)` | Type rules. |
| `connect(from, to, amount?)` | Create a connection (gain node, SVG, bookkeeping, undo). If the cable closes an audio loop, it is routed through a one-render-quantum `DelayNode` (`c.dly`, ≈ 3 ms) and the inspector shows "feedback (+3 ms)". |
| `audioPathExists(a, b)` | Is there an audio path from module `a` to module `b` (cables into jacks or audio-rate params)? Used by the feedback guard. |
| `setAmount(c, a)` | Apply depth (scaled into parameter units for modulation). |
| `disconnect(c)` | Remove the connection (records undo first). |
| `rewireParam(p, targets)` | Move modulation cables to new AudioParams (used by LFO waveform changes). |
| `updPlugs()` | Light up connected plugs. |
| `jackEl(j)`, `jackPos(o)`, `wpath(a, b)`, `drawWire(c)`, `layoutWires()` | Jack DOM and cable geometry. |
| `selectConn(c)` | Select a cable. |
| `startCable(e, jack)` | Interactive cable drag, including picking up an existing cable from an input. |
| `fireTrig(jack, t, vel, dur, note)` / `fireRel(jack, t, note)` | Dispatch trigger events; a throwing receiver is logged and skipped. |

### § 8–10 Inspector, navigation, clipboard, scheduler

| Function | Description |
|---|---|
| `showInsp()` | Rebuild the bottom bar (titles and labels are escaped). |
| `startMarquee(e)` | Rubber-band selection. |
| `fitView()` | Zoom to fit every module (0.28×–1.3×). |
| `isTextTarget(t)` | Is the event target an input/textarea/select/contentEditable? |
| `copyFragment()` / `pasteFragment(frag)` | Clipboard fragment build / place at the pointer. |
| `cloneBoundaryCables(mods, idMap, mode)` | Re-create cables crossing a duplicated selection. |
| `duplicateModules(mods, {cables})` | Duplicate to the right; one undo step. |
| `readTaps()` | Average each tapped jack's analyser buffer into `jack.cv`. |
| `pumpControl()` | `readTaps()` + apply all control-rate params. |
| `pumpSched(t, ahead)` | Call every module's `sched` (errors are swallowed). |
| scheduler interval | Every 22 ms: `pumpControl()` (so modulation keeps running in background tabs, where rAF stops), loop wrap, `pumpSched`, `songTick`. |

### § 11 Module definitions

`DEFS.osc`, `DEFS.poly`, `DEFS.drum`, `DEFS.filter`, `DEFS.crush`, `DEFS.echo`,
`DEFS.rev`, `DEFS.env`, `DEFS.lfo`, `DEFS.seq`, `DEFS.vca`, `DEFS.mix`, `DEFS.scope`,
`DEFS.out` and `DEFS.keys`; see § 5. Sequencer helpers:

| Function | Description |
|---|---|
| `seqDefaultSteps()` | 32 default steps (every 4th step on). |
| `seqPackSteps(S)` / `seqUnpackSteps(p)` | Columnar share codec for one pattern. |
| `m.isDefaultBank`, `m.fillBank`, `m.resetBank`, `m.setBank`, `m.refreshUI`, `m.stepDur`, `m.locate`, `m.recNote` | Per-instance SEQ API. Internally `drainQ()` advances the playhead queue from both `sched` and `draw`. |
| LFO internals | `rateHz()`, `remake(t, phase)`, `phaseWave(wave, phase)` (phase-shifted PeriodicWave from the spec's Fourier series), `relock()`, `m.setT`, `m.locate`, `m.free`. |
| REVERB internals | `buildIR()` (uses modulated values), `irKey()` (change detector), `queueIR()` (140 ms throttle). |
| `m.noteOn(raw, vel)`, `m.noteOff(raw)`, `m.allOff()` | Per-instance KEYS API. |

### § 12–14 Palette, render loop, serialisation

| Function | Description |
|---|---|
| `frame(t)` | Main rAF loop. |
| `rescaleCanvases()` | Re-rasterise every module canvas (`m.vizList`) when `devicePixelRatio` changes (monitor switch / browser zoom); modules with extra canvases implement `m.onDpr(ratio)` (SCOPE). |
| `migrate(d)` | Validate and upgrade schema (`MIGRATIONS`). |
| `modById(id)` | Module lookup. |
| `exportModule(m)` | Module → JSON (`id, t, n, x, y, c, p, s, mc`). |
| `cableRef(c)`, `refKey(ref)`, `findConnByRef(ref)`, `connectRef(ref, lookup)` | Id-based cable references (survive module re-creation). |
| `exportFragment(mods, {cables})` | Modules (+ internal cables) → fragment. |
| `serialize()` | Whole rack + tempo + song → patch. |
| `importFragment(frag, {keepIds, dx, dy, at})` | Fragment → live modules/cables; returns `{modules, conns, idMap, skipped}`. Ids are reused only if they look like `m<digits>`; each record's params/state are imported defensively. |
| `restoreTitle(m, name)` | Restore a saved title (only `"<TYPE> <n>"` is accepted) and advance the per-type counter. |
| `clearPatch()` | Remove all modules and reset the song. |
| `loadPatchData(d)` | Raw load (clear, song, tempo, import). |
| `deserialize(d)` | Undoable load plus fit view. |
| `loadBuiltin(name)` | Undoable built-in load. |
| `P_init`, `P_acid`, `P_drums`, `P_ambient` | Built-in racks. |

### § 15 Library & share

| Function | Description |
|---|---|
| `libRead()` / `libWrite(u)` | localStorage library I/O. `libRead` returns a null-prototype object, so any name (even `__proto__`) is an ordinary key. |
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
| internal: `b64.enc/dec`, `deflate`, `inflate` (capped), `pack(opt)`, `packSong(idx)`, `unpack(json)`, `sig4` | Codec pieces. Links are written as `v: 2`. `unpack` detects packed vs saved state by shape, so links from the earlier (buggy) encoder still open with their patterns. |

### § 16–21

| Function | Description |
|---|---|
| `rec` | Recorder state: `{on, node, blocks, tStart, endBeat, tEnd, startBar…}`. |
| `recStartFn()` | Arm the tap; in bars mode, start the transport if needed and pick the next bar line. |
| `recBlock(t0, L, R)` | Receive a time-stamped block; finish once the end time is covered. |
| `recEndTime()` | End of the take: fixed (manual/transport stop) or `transport.timeAt(endBeat)`. |
| `recStopFn()` | Manual stop (with a 1 s watchdog in case blocks stop arriving). |
| `recFinish(tEnd)` | Disconnect, crop to `[tStart, tEnd)`, encode with `encodeWavPCM(…, 16)` and download. |
| `setMidi(p, ch, cc)` | Attach a MIDI CC mapping. |
| `noteInput.*` | `listeners, on, off, panic, record, setQwerty, releaseQwerty, handleKeyDown, handleKeyUp, enableMidi, midiMsg, setLearn, armLearn, clearLearnTarget, cc`. `record(note, vel)` writes into every record-armed SEQ (used by QWERTY, MIDI and KEYS' on-screen keys). |
| `anyDialogOpen()` / `closeAllPanels()` | Keyboard helpers: shortcuts are inert while a dialog is open; Esc closes everything. |
| `song.*` | `bankFor(id, beat)`, `snapshot(prune)`, `restore(d)` (validated), `touch()`. |
| `applySongToTransport()` | Sync the loop region. |
| `songTick()` | Stop at the song end (song mode, no loop). |
| `songCommit(label, before)` / `songEdit(label, fn)` | Undoable song edits. |
| `songUI.toggle()` / `songUI.close()` / `songUI.tick()` | Arranger panel (internal: `fits`, `trimToLength`, `syncHead`, `clipEl`, `renderGrid`). |
| `withHeadlessRack(engine, fn)` | Swap in a hidden rack bound to another engine, then restore. |
| `renderOffline({bars, tail, stems, onProgress})` | Offline bounce → `{files:[{name, L, R}], sampleRate, seconds, capped}`. |
| `encodeWavPCM(L, R, sr, bits)` | 16/24-bit PCM or 32-bit float WAV. |
| `makeZip(files)` | Store-only ZIP writer (CRC-32). |
| `downloadBlob(blob, name)` | Trigger a download. |
| `exportUI.open()` / `exportUI.close()` | Bounce dialog (internal `go()`); `close()` refuses while a render runs. |
| `boot(which)` | `startAudio()` → engine init → load patch → autoplay. Ignored if already booted. |

---

## 14. Changelog & known issues

### 14.1 Fixed in this revision

A full review found 18 critical/high/medium issues, and all of them are fixed. Each fix
was re-verified in headless Chromium 141, both over `http://` and from `file://`.

| # | Area | Problem | Fix | Verified result |
|---|---|---|---|---|
| C1 | Engine | `mkNoise()` was a placeholder, so every noise source (OSC white/pink, DRUM noise layers, the whole **clap** engine) was silent. | Real white noise plus pink noise (Paul Kellet filter), normalised to a 0.95 peak. | Both buffers peak at 0.95; clap is audible. |
| C2 | OSC | The `Trig` input called undefined `fire`/`rel`. Patching a gate into it froze the sequencer, which then re-fired one step about 42×/s. | The broken jack is removed. Its slot is now the FM input (H4). `fireTrig`/`fireRel` also catch per-receiver errors, so one faulty module can never stall a sender. | With a deliberately throwing receiver, the SEQ keeps time (9 steps/s). |
| C3 | ENV | No `onRel`, so notes from KEYS/MIDI/piano never released. | `onRel: t => rel(t)` on the ENV Trig jack. | Level 0.5 while held → 0 after note-off. |
| C4 | Share | Links were labelled v1 but packed with the v2 codec, so **all sequencer patterns were lost**. | Links are written as `v: 2`. The decoder detects packed vs saved state by shape, so links made by the old encoder still open correctly. | Patterns survive the round trip, including a legacy link. |
| H1 | Security | Patch names, module titles and file names reached `innerHTML` (library rows, toasts, inspector, song lanes), allowing script injection from a crafted `.json`. | New `ht()` (text) and `esc()` helpers. `toast()` is text-only. Library rows, metadata, inspector and song labels are escaped. `restoreTitle` only accepts `"<TYPE> <n>"`. | An injected `onerror` payload fires 0 times (previously 6). |
| H2 | Share/Load | `DEFS['constructor']` etc. were truthy. A hostile link threw mid-load, wiped the rack and left no undo entry. | `DEFS` has a null prototype. `addModule` tears down and skips a module whose `build()` throws. Per-record import is defensive, and only `m<digits>` ids are reused. `history.replace()` rolls back to the previous rack if the load throws. Boot falls back to an empty rack if a shared patch fails. | Hostile link: invalid modules are skipped and the load is undoable (10 → 1 → 10). A forced mid-load exception restores the original 10 modules. |
| H3 | KEYS | `m.def.free = …` overwrote the shared definition, so deleting one KEYS released another's notes. | Per-instance `m.free`, called by `removeModule` alongside `def.free`. | Deleting KEYS #1 calls only KEYS #1's `allOff`. |
| H4 | OSC | The **FM Amt** knob scaled a gain node with no input. | New `FM` input jack (linear FM, depth set by FM Amt in Hz). | Zero crossings 1164 → 1677 with FM applied. |
| M1 | Toolbar | A second `#clr` handler replaced the undoable one, so one clear made 13 undo steps. | The duplicate handler is removed. | Clear = 1 undo step. |
| M2 | REVERB | Size/Decay/Damp/Diffuse modulation did nothing: the debounce restarted every frame, and `buildIR` read the un-modulated values. | `buildIR` uses `effective()`. Rebuilds are throttled (≤ 1 per 140 ms) and skipped for inaudible changes. | 12 distinct IRs in 2 s under LFO modulation (previously 1). |
| M3 | Params | OSC/POLY **Pitch** used the multiplicative `cents` mapping on a cents-valued knob, so the readout showed "0¢" while at +1200¢. | `cents:1` removed from both Pitch knobs (linear `base + cv·4800`). | A CV of 0.25 reads "1200¢". |
| M4 | ENV | Loop mode never played the release. | The release is scheduled for every timed gate, including loop cycles. | About 55 % of each loop cycle is spent in the release segment. |
| M5 | LFO | Synced LFOs ignored BPM changes, weren't bar-aligned, and leaked their oscillator on delete; retrigger left an output gap. | `m.setT` follows the tempo. `m.locate` restarts the oscillator at the transport-implied phase, using a phase-shifted `PeriodicWave`. `m.free` now runs on delete. The old oscillator plays until the new one starts. | 7 → 14 zero crossings per 2 s when the tempo doubles; value −1.00 at beat 3 for a 1/1 sine started at beat 2. |
| M6 | ECHO | Feedback up to 1.15 grew without bound; 🎲 could land there. | Soft saturator in the feedback loop (unity for small signals, ±1.5 ceiling). New `rnd:[lo,hi]` param option; echo Fb randomises within 0…0.85. | Fb 1.15 for 8 bars: stem peak 2.19 and finite; randomise max 0.847. |
| M7 | Output | Master and OUT "limiters" were plain compressors, and bounces peaked at +1.3 dBFS. | `mkSoftClip()` brick-wall soft clipper after the master compressor and after each OUT compressor. | Worst-case bounce peak 0.82–0.89 (previously 1.165). |
| M8 | Bounce | Ignored the master slider, started 80 ms late, silently capped at 600 s, and had undocumented stem tap points. | Offline master gain follows the slider. `transport.play(from, delay)` is used with delay 0 offline. The sequencer treats `nextT === null` (not 0) as "unset". Capping is reported in the dialog and a toast, and the dialog text documents the stems. | Peak ratio 2.01 for a 2× slider change; first sound at ~18 ms (compressor lookahead plus attack) instead of 92 ms; `capped: true` reported. |
| M9 | Song | Loading a built-in or clearing kept the old song mode/loop/length. | `clearPatch()` resets the song; undo restores it. | After `loadBuiltin`: song off, 8 bars. |
| M10 | Timing | Control-rate modulation ran only in rAF (which stops in background tabs), and the SEQ playhead queue only drained while the module was visible. | `pumpControl()` also runs on the 22 ms scheduler. `drainQ()` runs from `sched` as well as `draw`. | Queue length is 1 after 2 s of playback while collapsed. |

Compatibility notes:

* Patches that had a cable into the old OSC `Trig` jack load without that cable (the
  jack never worked).
* Share links created before this fix still decode, and now *with* their patterns.

### 14.2 Low-severity fixes

| # | Problem | Fix | Verified result |
|---|---|---|---|
| L1 | Debug `console.log` calls in the palette code. | Removed. | No console output while booting any preset. |
| L2 | Malformed help HTML (unclosed/nested `<li>`, orphan text). | Rewritten into well-formed *mouse / editing / playing & song* lists; also documents the feedback guard and bar-aligned recording. | No nested `li`, no stray text nodes. |
| L3 | Dead or duplicated code. | Removed `setPlay()`, the empty AudioParam doc block, the duplicate per-frame `posEl` write and the recorder's duplicate `encodeWav`. Each module's engine destructure is trimmed to the names it uses. The meter reuses one buffer. | Syntax check plus the full regression suite. |
| L4 | Stale section index and development-note comments. | Section index covers §§ 1–21 (with 5.5/6.5/9.5/9.6/15.5). The header documents the note protocol. "Patch/Step/Refactor" comments are replaced with section references. | – |
| L5 | `quantN` ignored neighbouring octaves. | Candidates include ±12; exact ties resolve downward. | PentMaj pc 11 → 12; PentMin pc 11 (tie) → 10. |
| L6 | Swing used pattern-index parity, so the feel flipped with odd lengths. | Swing uses an absolute grid counter `st.abs`. `locate()` derives it from the beat, and odd grid steps are placed swung-late. | 3-step pattern, swing 0.5: gaps 188/63/188/63… ms. |
| L7 | Acid preset VCA never closed (≥ 0.63). | Vel → VCA cable removed; ENV → VCA at 1.0; accents via ENV Vel = 0.5. | VCA minimum 0.000. |
| L8 | Fb knob overrode Echo Freeze. | Fb's `set()` is ignored while frozen; unfreezing restores the knob value. | Feedback stays at 1.0 after moving Fb while frozen. |
| L9 | On-screen KEYS notes weren't recorded. | New `noteInput.record()` used by QWERTY, MIDI and the on-screen keyboard. | Clicking C4 in step-record writes step 1. |
| L10 | Recorder started at the click, used ScriptProcessor on the main thread, and wasn't sample-exact. | New `rec-tap` AudioWorklet with time-stamped blocks (ScriptProcessor only as fallback). Bars mode starts on the next bar line, or with the transport if stopped; the end is defined in beats, and the take is cropped sample-exactly. | 2 bars @ 120 BPM = 4.0000 s; fallback: 1 bar @ 122 BPM = 1.967 s, first hit at the same offset as a bounce. |
| L11 | Delay-less feedback loops are muted by spec-compliant browsers. | `connect()` detects loop-closing cables (`audioPathExists`) and inserts a one-quantum `DelayNode`; toast plus inspector hint. | Self-loop and downstream loop get 2.9 ms delays; normal cables don't. |
| L12 | UI details. | Shortcuts are inert while a dialog is open. Esc closes everything (bounce, song, even from a text field). Knob drags are incremental (no jump on Shift). Canvases re-rasterise on `devicePixelRatio` changes (`rescaleCanvases`, SCOPE `m.onDpr`). Worklets fall back to a `data:` URL (work on `file://`). The AudioContext is created on the first click (`createEngine(null)` + `startAudio()`). The library uses null-prototype objects (`__proto__` is a valid name). | Space ignored under help; Esc closes help/bounce/song; Shift toggle mid-drag: 0.200 → 0.201 (no jump); canvas 414 → 828 px at DPR 2; worklet on `file://`: yes; `ENGINE.ac === null` before the click; `__proto__` patch stored and read back. |

### 14.3 Known limitations and design notes

* **REVERB modulation is stepped.** Size/Decay/Damp/Diffuse changes rebuild the
  impulse response (at most ~7×/s, with fresh noise each time). Smooth sweeps need
  PreDly/LowCut/HiCut/Mix.
* **LFO relocking creates oscillators.** Every play/seek/loop relock of a synced LFO
  creates a new `OscillatorNode` (cheap, but visible in profilers).
* **Master soft clipper.** It adds gentle saturation above about −2 dBFS.
* **Feedback guard is conservative.** It assumes every module passes its inputs to its
  outputs, so it may add its 3 ms delay to a cable that doesn't strictly need it. It
  also can't see loops that only become audio-rate later (e.g. an LFO Rate cable
  becoming audio-rate when the LFO switches from a random to a periodic wave).
* **Compressor latency.** The per-OUT and master compressors add a few milliseconds of
  lookahead latency, visible as ~18 ms before the first hit in bounces/recordings of
  drum patches. Timing *between* events is unaffected.
* **Background tabs.** Browsers may throttle the 22 ms scheduler timer in hidden
  tabs. Tabs that are playing audio are normally exempt, but this is
  browser-dependent.

### 14.3 Testing

There is no test suite in the repo. All of the verification above was done by driving
the page in headless Chromium with Playwright, using `page.evaluate` against the app's
globals (`modules`, `connect`, `renderOffline`, `SHARE`, `history`…). Useful checks to
keep:

* Boot each preset from `file://` and `http://` and assert no console output.
* Bounce two bars and assert `0 < peak ≤ 0.99`.
* Round-trip a share link and compare SEQ patterns.
* Load a hostile link or file and assert the rack survives and nothing executes.
* Record 2 bars at a known tempo and assert the WAV length.
* Use real mouse events (Playwright `page.mouse`) for knob drags: synthetic
  `PointerEvent`s can't take pointer capture.
