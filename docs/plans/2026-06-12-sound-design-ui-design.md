# Sound-Design UI: Design Spec

**Date:** 2026-06-12
**Branch:** `feature/tone-editing-ui` (rebased onto `perf/emulator-optimizations`)
**Goal:** Make the JV-880's sound-design power reachable from the Move hardware: common parameters (filter, envelopes, levels) on knobs with minimal menu nesting, full per-tone editing one level down, plus rhythm sets and (later) multi-slot "clones."
**Explicitly out of scope:** emulated-LCD/front-panel mode (rejected); per-clone audio routing (hardware mixes all parts to one stereo pair).

## Ground truth this design builds on

- The C plugin (v2, instance-based) already implements get/set for ~68 per-tone params (`nvram_tone_*`), 8 patch-common params (`nvram_patchCommon_*`), part params (`sram_part_*`), and 6 cross-tone macros — all verified against the JV-880 SysEx map (`PARAMETER_AUDIT.md`). Writes go to NVRAM **and** DT1 SysEx into the firmware (live + readback consistent).
- `ui_hierarchy` in `module.json` is a 1-level stub; `ui.js` on main already renders arbitrary nested hierarchies (list levels, child_count levels, knob arrays per level). **Most of Phase B is data, not code.**
- The upstream `menu-driven-ui-redesign` branch has a complete menu taxonomy (Tone → Wave/Pitch/Filter/Amp/LFO) worth mining for structure/labels; its DSP approach (SysEx-only, bypasses NVRAM) is NOT used — main's nvram+SysEx path is the right one.
- 64 user save slots exist in the DSP (`save_to_slot_N` / `do_save_to_slot`).

---

## Phase A — Performance page (root knobs)

Root level of `ui_hierarchy` gets 8 knobs:

| Knob | Param | Backing |
|---|---|---|
| 1 | Cutoff (macro) | all 4 tones' `cutofffrequency` |
| 2 | Resonance (macro) | all 4 tones' `resonance` |
| 3 | Attack (macro) | all 4 tones' `tvaenvtime1` |
| 4 | Decay (macro) | all 4 tones' `tvaenvtime2` |
| 5 | Release (macro) | all 4 tones' `tvaenvtime4` |
| 6 | TVF Env Depth (macro) | all 4 tones' `tvfenvdepth` |
| 7 | Reverb Level | `nvram_patchCommon_reverblevel` (absolute, exists) |
| 8 | Chorus Level | `nvram_patchCommon_choruslevel` (absolute, exists) |

All 8 appear in `chain_params` so the Shadow UI/chain page gets them too.

### Macro semantics: absolute-anchored, offset-preserving

Josh's decision: **absolute** — what you see is what the patch is, edits persist in the patch temp area (no silent reset), saveable. But a naive "set all 4 tones to knob value" would flatten deliberate inter-tone differences (e.g. tone 2 darker than tone 1). Spec:

- **Display value** = the parameter value of the *anchor tone* (lowest-numbered enabled tone).
- **Turning the knob** computes `delta = new - anchor_value` and applies `clamp(tone_value + delta)` to every enabled tone — anchor tracks the knob exactly; other tones move in lockstep, preserving offsets.
- **Persistence**: writes go through the normal `nvram_tone_*` path (NVRAM + SysEx), so values survive until patch reload like any tone edit, and `do_save_to_slot` captures them.
- **Patch change**: knob re-reads the anchor tone's stored value — no hidden state, nothing to reset.

### Switchability requirement (the abstraction)

All macro behavior funnels through one function pair in the C plugin:

```c
static int  macro_read (inst, const macro_def_t *m);              /* -> display value */
static void macro_write(inst, const macro_def_t *m, int value);   /* apply semantics */
```

with a single compile-time (or `set_param`-able, see below) selector:

```c
typedef enum { MACRO_ABSOLUTE_ANCHORED, MACRO_RELATIVE_RESET } macro_mode_t;
#define MACRO_MODE_DEFAULT MACRO_ABSOLUTE_ANCHORED
```

`MACRO_RELATIVE_RESET` preserves today's behavior (offset table, cleared on program change). Implementation keeps the existing offset code intact behind this switch rather than deleting it. Expose `set_param("macro_mode", "absolute"|"relative")` (not surfaced in UI yet) so A/B testing needs no rebuild. **Acceptance: switching modes is a one-string change, no UI or table changes.**

---

## Phase B — Tone editing (nested hierarchy + save)

### Hierarchy (rendered by existing ui.js machinery; defined via `build_ui_hierarchy()` in C so it can be model/mode-aware)

```
root (browse + 8 performance knobs, as Phase A)
├── Edit: Common      — patch level, pan, analog feel, bend range, portamento, reverb/chorus type+params
├── Edit: Tone 1..4   — child_count level (tone tabs, TONE LEDs mirror selection)
│   ├── [knobs at tone level]: cutoff, resonance, TVA attack, TVA decay,
│   │                          TVA sustain (tvaenvlevel3), TVA release, level, pan
│   ├── Wave          — wavegroup, wavenumber, FXM switch/depth, tone switch
│   ├── Pitch         — coarse, fine, keyfollow, pitch env (depth, T1-4/L1-4)
│   ├── Filter        — mode (Off/LPF/HPF), cutoff, resonance, keyfollow,
│   │                   TVF env (depth, velo, T1-4/L1-4)
│   ├── Amp           — level, pan, velo sense/curve, TVA env (T1-4/L1-4), delay mode/time
│   ├── LFO 1 / LFO 2 — form, rate, delay, fade, offset, sync, pitch/TVF/TVA depths
│   └── FX Sends      — dry, reverb send, chorus send
└── Save              — pick user slot 1..64, confirm (uses existing do_save_to_slot)
```

- Max depth: 3 (root → Tone N → section). Every level defines a `knobs` array so the 8 encoders are always live; section pages put that section's most-tweaked params on knobs 1–8 in manual order.
- All values are **absolute per-tone** `nvram_tone_*` keys — already implemented in C; this phase is hierarchy data + UI labels + value formatting (e.g. cutoff 0–127, filter mode enum names, env time display).
- Performance mode: existing part-level editing stays as is; per-part tone drill-down is out of scope for this phase.
- Screen-reader: `announceMenuItem(label, value)` on knob/jog edits, `announceView` on level changes (shared helpers, matches platform convention).

### Mining, not merging

Take from `upstream/menu-driven-ui-redesign`: the section taxonomy, label strings, param orderings. Do not take: its DSP changes, its SysEx-direct send path, its JS Map value cache (readback bug). Risk note: its bank/patch accessor params (`bank_N_name` etc.) are not needed — main's browser already works.

---

## Phase C — Rhythm sets

**Requirement (Josh):** load rhythm sets directly like presets and play them on whatever MIDI channel is routed to the module — not locked to ch10/rhythm-part convention.

Design sketch (needs one investigation task at implementation start):

1. **Loading:** rhythm sets live in ROM2/expansions alongside patches. Add them to the browsable list as a "Rhythm" bank (or preset-type filter), loaded via the same patch-temp mechanism if the firmware supports rhythm in patch mode, else via performance mode with the rhythm part soloed. ← *investigation decides which mechanism; the emulator's `MCU_BUTTON_RHYTHM` front-panel path and the JV-880 SysEx rhythm-setup area (`00 07 xx xx` region) are the two candidate routes.*
2. **Channel freedom:** the module already owns MIDI before the firmware sees it (`v2_on_midi` → queue → `postMidiSC55`). When a rhythm set is active, rewrite the incoming channel nibble to whatever channel the firmware expects for the rhythm part. The user-facing behavior: notes on the module's routed channel just play drums. Zero firmware modification.
3. **Knobs in rhythm mode:** keep patch-common FX knobs (7–8); knobs 1–6 TBD after investigation (per-key editing is a deep rabbit hole — v1 ships browse+play+FX, per-key editing deferred).

**Acceptance:** pick a rhythm set from the browser, hit pads on the module's normal channel, drums play; FX knobs work; patch/rhythm switching is seamless.

---

## Phase D — Clones (DEFERRED pending audio-routing investigation)

**Status (2026-06-13):** Deferred by Josh until we test what per-clone audio routing would take. The control-plane design below stands, but clones-without-separate-audio aren't worth shipping; the gate is the investigation:

**Per-part audio bus investigation (gate for this phase):** the hardware mixes all parts to one stereo pair, but the EMULATOR owns the mixer — per-voice accumulation happens in pcm.cpp. If the firmware's voice-allocation state (slot → part mapping) can be read from emulated RAM/registers, the voice loop could accumulate into per-part stereo buses, giving each clone slot its own audio (beyond what real hardware could do). Tasks: locate the firmware's voice-allocation table (RAM dump diffing while keying known parts), prototype slot→part tagging in the PCM mix, measure CPU cost of N buses, and define how clone render_blocks pull their bus. If the mapping can't be recovered reliably, clones stay deferred.

### Original control-plane design (kept for when the gate passes)

**Requirement (Josh):** schwung's MIDI routing is inflexible; want clone modules loadable in other chain slots that share ONE emulator instance — each clone bound to its own part: unique MIDI channel, preset, params.

Architecture (all within Plugin API v2; every slot dlopens the same `dsp.so` into the same process, so process-globals are shared):

- **Primary election:** first `create_instance` becomes primary (owns the emulator, switches to Performance mode); subsequent instances detect the primary via a process-global registry (mutex-guarded) and become clones bound to the next free part (2..8). `destroy_instance` of a clone frees its part; of the primary → all clones return silence and mark "orphaned" (v1: require the primary slot; document the limitation).
- **MIDI:** clone's `on_midi` rewrites channel → its part's channel, queues into the primary's MIDI FIFO.
- **Params/preset:** clone's `set_param`/`get_param` map to `sram_part_*` for its part (patch select, level, pan, tune, key range) — already implemented in C.
- **Audio:** the primary's `render_block` outputs the full mix; clones output silence. Documented loudly: per-clone FX chains affect nothing (single stereo bus in hardware). The clone's value is routing + control, not audio separation.
- **Module packaging:** clones are the same module (a `clone_of` capability or a tiny variant `module.json` with the same dsp.so) — decide at implementation; prefer one module that self-detects to avoid catalog duplication.
- **CPU:** one emulator total (~16% thread on CM5) instead of N — the whole point.

This phase is experimental by nature; it ships behind its own branch and gets a design review of its own once A–C are proven.

---

## Cross-cutting

- **Param metadata single source of truth:** one C table per param: key, display name, min/max, enum labels, SysEx index, NVRAM offset — `chain_params` JSON, `ui_hierarchy`, and value formatting all generate from it. No duplicated ranges in JS.
- **State serialization:** `state` JSON version bump; macro mode + edited patch temp included so chain autosave/restore round-trips edits.
- **Perf stats:** flip `JV880_PERF_STATS` to 0 in the release build of this feature branch (instrumentation stays available for dev).
- **Validation:** the WAV harness covers the engine; UI phases need on-device manual test scripts (extend `TEST_PLAN_PARAMS.md` per phase). Param writes are already exercised by the existing audit.
- **Upstream posture:** A+B are clean upstream candidates (pure additive UI + one macro-semantics change, switchable back). Rhythm sets likely upstreamable. Clones may stay fork-only unless charlesvestal wants them.

## Sequencing & estimates

| Phase | Contents | Est. |
|---|---|---|
| A | Root knobs, absolute-anchored macros behind switch, chain_params | 1–2 days |
| B | Hierarchy data, tone pages, value formatting, save-to-slot UI, screen-reader | 3–5 days |
| C | Rhythm sets (investigation + browse/play/channel-rewrite) | 2–4 days |
| D | Clones (registry, part binding, lifecycle) | 1 week+, own design review |

Open questions to resolve during implementation (not blockers): rhythm-set load mechanism (C.1); clone packaging (D); whether macro knobs should skip disabled tones in anchor selection (default: yes).
