# CG — Concrete Guitar Toolkit (JSFX) — 21 modules

Install: copy all `.jsfx` files into `REAPER/Effects/CG/` (Options → Show REAPER resource path), then Actions → "Rescan" or just add via FX browser ("CG ...").
Every file is self-contained (no imports). Modules are mono-in (auto-picks the live channel), stereo-out, insert-style with Wet/Dry.

## Suggested chain
`CG Guard` (first) → `CG Analyzer` → any modules in any order → `CG Guard` (last)

## Modules
| Module | What it does to the live guitar |
|---|---|
| **Guard** | NaN/inf scrub, DC block, 1.5 ms look-ahead brick-wall limiter, hard ceiling, runaway-mute (fades out if >20 dB gain reduction persists 250 ms; auto-recovers). Use first AND last. |
| **Analyzer** | Pass-through. Publishes level, onsets, brightness, noisiness, flux, pitch (+confidence), onset density on a shared bus (4 buses). |
| **Splicer** | Captures slices of what you just played (or the N ms after a trigger), loops/stutters/reverses/repitches them. Triggers: own onset detector, analyzer bus, manual button, MIDI note (note = pitch). |
| **Grains** | Live granulator reading your own recent audio: position, spray, size, density, pitch spray, reverse, freeze. |
| **Warp** | Varispeed/pitch warp, reverse, tape-stop/start, random slips, wow, stereo detune. |
| **RingShift** | Ring mod + Hilbert frequency shifter (with bounded feedback). Carrier fixed, pitch-tracked, or envelope-driven; morphable waveform. |
| **Resonator** | 8 tuned feedback combs excited by the guitar; ratio sets, detune, RT60 decay, optional pitch-tracked root. |
| **Rupture** | Wavefold, bit-crush, sample-rate reduction, rectify, asymmetric clip, tone filters. |
| **EnvShape** | Transient shaper, volume swell on onsets, gate with hold, random cut-up. 12 ms look-ahead (latency reported to REAPER). |
| **Echo** | 4-tap slewing tape-style delay, drift, saturating filtered feedback, HOLD freeze, ducking. |
| **Spectral** | 1024-pt FFT: spectrum capture/hold, blur, peak gate, bin scatter, stretch, phase randomize. 1023 samples latency (reported). |

### Creative set (added)
| Module | What it does |
|---|---|
| **Reverb** | 8-line FDN (Householder matrix), modulated lines, damping, pre-delay, true FREEZE (unity loop, input muted). Analysis → decay/damping. |
| **Smear** | 8-stage modulated allpass diffusion per channel; melts attacks into wash; bounded wash-feedback. |
| **Glitch** | Random/onset-triggered micro-repeats of your last 2 s: random slice length, repeats, accelerating shrink, reverse, pitch variance, dry ducking. |
| **Drift** | Six delay voices on smooth random walks: unstable detune ensemble / tape warble. |
| **Vowel** | 3-formant vocal filter morphing A-E-I-O-U; driven by envelope or analysis. |
| **Shimmer** | Pitch-shifting feedback delay: every repeat is shifted again (cascading octaves / fifths / downward). |
| **Space** | Pan/Haas/darkening mover driven by LFO, attacks (random jumps) or any analysis stream. |

### Modulation, ecology, networks (added)
| Module | What it does |
|---|---|
| **Modulator** | 4-lane modulation source. Every lane picks one of 16 algorithms and publishes a 0..1 stream on the bus; any processing module can then use it via **Mod Source = Mod 1 … Mod 4**. *Deterministic:* Sine, Triangle, Saw Up/Down, Square (pulse width), Step Pattern (seeded 8-step), Onset Envelope (from your attacks). *Stochastic:* Sample+Hold, Smooth Noise, Random Walk, Poisson Bursts, Random Gate. *Chaotic:* Lorenz, Rossler, Logistic Map, Henon Map (Shape = chaos parameter). Per lane: rate, shape, smoothing, bipolar depth, offset. **Lane Link** lets each lane's rate follow the previous lane (ring) and **Analysis → Rates** lets your playing speed up/slow down all lanes. **Seed + RESTART** make it repeatable. Audio passes through. |
| **Ecosystem** | Di Scipio-style feedback ecology: 2–6 "cells" (delay + resonant filter), coupled in a ring and to themselves. Homeostasis regulates each cell's loop gain toward a target level (the system self-starts from tiny spontaneous activity and breathes around that level), predation lets stronger neighbours suppress weaker ones, adaptation tunes cell filters to harmonics of the sound's own dominant frequency, mutation randomly drifts tunings/delays, and your guitar is the perturbation from outside. Bounded by saturators + watchdog. |
| **DelayNet** | Up to 8 delay nodes with routable topology: Ring, Ping-Pong Pairs, Star, Cascade, Dense, Random Sparse (seeded), Gravity. Time patterns Even/Golden/Primes/Random, per-node filters with spread, gain-neutral saturation, drift, slewed times, input injection patterns, HOLD freeze. Every row of the routing matrix sums to ≤ the feedback amount, so loop gain is provably < 1 (HOLD = 0.999). Topology/feedback changes glide, never click. |

**Bus layout** (per bus 1–4, 16 slots): 0 Analyzer heartbeat, 1–6 analysis streams, 7 pitch confidence, 8 onset density, 9 pitch Hz, 10 onset counter, 11–14 Modulator lanes 1–4, 15 Modulator heartbeat. If the Analyzer or Modulator is missing/bypassed for >0.4 s, their streams read as 0 in every module.

## Analysis → processing (no resynthesis)
Every processing module has **Mod Source** (Level / Onset / Brightness / Noisiness / Flux / Pitch from the Analyzer bus) + depth sliders on musically useful targets. If no Analyzer is present, mod = 0 (module still works). Use the same **Analysis Bus** number on Analyzer and modules; use different numbers per track if you run several guitars.

## Safety design
- All user functions are NaN-safe (`lim`, `fin`, `sat`); non-finite input is scrubbed to 0.
- Every feedback loop is bounded by soft clipping and gain < 1; every module has a watchdog that triggers a **fade → clear all buffers → fade-in** panic.
- Every module ends in a soft ceiling at +6 dB; Guard makes the final ceiling absolute.
- All parameters are smoothed; no waveform/mode hard switches (morphs and crossfades instead).
- No unbounded loops; all buffer indices wrapped/clamped; memory sizes capped for sample rates up to 192 kHz.
- `PANIC` button on each module clears its state manually.

## Testing done (headless JSFX interpreter, ysfx)
21 modules × 8 hostile inputs (plus full-chain Guard→Analyzer→Modulator→module→Guard integration with every module taking Mod 1 at maximum depth) (plucks, full-scale noise, silence, ±60 single-sample spikes, loud sine, hot clipped guitar, NaN/±inf injection, DC offset) × default and extreme settings: zero non-finite output samples, output never above +6 dB. Also tested at 44.1/96/192 kHz and block sizes 1–4096. DSP checked numerically: Warp +12 st = exactly ×2 frequency; RingShift ±200 Hz shift accurate with ~50 dB image rejection; Spectral reconstructs input exactly.
Not tested: real REAPER UI/GFX, real-time CPU in REAPER's JIT (should be lighter than the test interpreter), MIDI routing in your project.
