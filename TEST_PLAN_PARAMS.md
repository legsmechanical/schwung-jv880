# Mini-JV Parameter Test Plan

This test plan verifies the NVRAM offset corrections for Patch mode (tone) and Performance mode (part) parameters.

## Prerequisites

1. Build and install the updated module:
   ```bash
   ./scripts/build.sh && ./scripts/install.sh
   ```
2. Restart Schwung on the device
3. Load Mini-JV in Signal Chain or standalone

---

## Part 1: Patch Mode Parameters (Tone Editing)

### Setup
1. Switch to **Patch Mode** (set mode = "Patch")
2. Select a patch with obvious tonal character (e.g., a bright pad or bass)
3. Select a tone using Track buttons 1-4

### TVF (Filter) Parameters

| Parameter | Test Procedure | Expected Result |
|-----------|----------------|-----------------|
| **Cutoff Frequency** | Turn knob mapped to cutofffrequency | Brightness changes - higher values = brighter, lower = duller |
| **Resonance** | Turn knob mapped to resonance | Peak/emphasis at cutoff increases - creates "wah" quality |
| **Filter Mode** | Cycle through filtermode | Off = no filter, LPF = low-pass (removes highs), HPF = high-pass (removes lows) |

### TVA (Amplitude) Parameters - CRITICAL (These were fixed)

| Parameter | Test Procedure | Expected Result |
|-----------|----------------|-----------------|
| **Level** (was at 68, now 67) | Adjust level parameter | Volume of selected tone increases/decreases |
| **Pan** (was at 69, now 68) | Adjust pan parameter | Sound moves left (0) to right (127), center at 64 |
| **TVA Env Time 1** (was at 75, now 74) | Adjust tvaenvtime1 | Attack time changes - higher = slower attack |
| **TVA Env Time 2** (was at 77, now 76) | Adjust tvaenvtime2 | Decay time changes - affects how sound fades after attack |
| **TVA Env Time 3** (was at 79, now 78) | Adjust tvaenvtime3 | Sustain/release time changes |
| **TVA Env Time 4** (was at 81, now 80) | Adjust tvaenvtime4 | Release time changes - how long sound rings after key release |
| **Dry Level** (was at 82, now 81) | Adjust drylevel | Dry/direct signal level changes |
| **Reverb Send** (was at 83, now 82) | Adjust reverbsendlevel | Reverb amount increases/decreases |
| **Chorus Send** (was at 84, now 83) | Adjust chorussendlevel | Chorus amount increases/decreases |

### Pitch Parameters

| Parameter | Test Procedure | Expected Result |
|-----------|----------------|-----------------|
| **Pitch Coarse** | Adjust pitchcoarse | Pitch shifts in semitones (obvious pitch change) |
| **Pitch Fine** | Adjust pitchfine | Subtle pitch change (cents) - useful for detuning |

### Switch Parameters (Enum Labels)

| Parameter | Test Procedure | Expected Result |
|-----------|----------------|-----------------|
| **Reverb Switch** | Toggle reverbswitch | Display shows "Off" or "On", reverb enables/disables |
| **Chorus Switch** | Toggle chorusswitch | Display shows "Off" or "On", chorus enables/disables |

---

## Part 2: Performance Mode Parameters (Part Editing)

### Setup
1. Switch to **Performance Mode** (set mode = "Performance")
2. Load a multi-part performance
3. Use Step buttons 1-8 to select different parts

### Part Parameters

| Parameter | Test Procedure | Expected Result |
|-----------|----------------|-----------------|
| **Part Level** | Adjust via Performance encoder macro | Selected part's volume changes |
| **Part Pan** | Adjust via Performance encoder macro | Selected part pans left/right |
| **Coarse Tune** | Adjust coarse tune parameter | Part pitch shifts in semitones |
| **Fine Tune** | Adjust fine tune parameter | Part pitch shifts slightly (cents) |
| **Key Range Low** | Adjust key range low | Part only sounds above this note |
| **Key Range High** | Adjust key range high | Part only sounds below this note |
| **Velocity Range** | Adjust velocity parameters | Part responds to different velocity ranges |

---

## Part 3: Verification Methods

### Method A: Auditory Verification
1. Play sustained notes while adjusting parameters
2. Listen for expected audio changes
3. Confirm changes are smooth and continuous (not glitchy)

### Method B: Visual Verification (if using debug logging)
1. Enable verbose logging in plugin
2. Watch for NVRAM writes at correct offsets
3. Watch for SysEx messages with correct indices

### Method C: State Persistence
1. Adjust several parameters
2. Save the state
3. Reload the patch/performance
4. Verify parameters restored to adjusted values

---

## Part 4: Regression Checks

Ensure these still work correctly:

| Feature | Test |
|---------|------|
| Preset browsing | Can cycle through patches |
| Bank navigation | Can navigate expansion banks |
| Tone selection | Track buttons 1-4 select tones in Patch mode |
| Part selection | Step buttons 1-8 select parts in Performance mode |
| Mode switching | Can switch between Patch and Performance modes |
| Octave transpose | Octave shift works correctly |

---

## Known Correct Values

From jv880_juce Tone struct (84 bytes per tone):

| Parameter | NVRAM Offset | SysEx Index |
|-----------|--------------|-------------|
| tvfCutoff | 52 | 74 |
| tvfResonance | 53 | 75 |
| tvaLevel | 67 | 92 |
| tvaPan | 68 | 94 |
| tvaEnvTime1 | 74 | 105 |
| tvaEnvTime2 | 76 | 107 |
| tvaEnvTime3 | 78 | 109 |
| tvaEnvTime4 | 80 | 111 |
| drySend | 81 | 112 |
| reverbSend | 82 | 113 |
| chorusSend | 83 | 114 |

---

## Troubleshooting

**Parameter has no effect:**
- Verify the correct tone/part is selected
- Check if the parameter makes sense for the current patch (e.g., filter mode might be "Off")
- Ensure the emulator has finished warming up

**Wrong parameter changes:**
- This indicates NVRAM offset is still incorrect
- Compare against jv880_juce dataStructures.h

**Sound glitches when adjusting:**
- SysEx message timing may need adjustment
- Check if emulator is keeping up with parameter changes

---

## Phase A — Performance-page macros (absolute-anchored)

On-device manual test checklist for the 8 root performance knobs (Cutoff, Resonance,
Attack, Decay, Release, TVF Env macros + Reverb, Chorus levels). Default macro mode is
`absolute` (MACRO_ABSOLUTE_ANCHORED).

1. **Knob shows patch value on load.** Select a patch. Each macro knob (1–6) must display
   the *anchor tone's* value — the lowest-numbered enabled tone (toneswitch On). Confirm
   Cutoff reads the same as Tone-1 `cutofffrequency` (when Tone 1 is enabled). Reverb/Chorus
   knobs (7–8) show the patch-common levels.

2. **Turn Cutoff → all enabled tones move, offsets preserved.** Pick a patch where tones
   differ (e.g. Tone 2 darker than Tone 1). Turn the Cutoff macro up by N. Verify every
   *enabled* tone's `cutofffrequency` increased by N (clamped 0–127) and inter-tone
   differences are preserved (anchor tracks the knob exactly). Disabled tones (toneswitch
   Off) must NOT change. Confirm audibly the timbre opens up.

3. **Switch patch → knob re-reads, no reset artifacts.** Change to another patch. Each macro
   knob immediately reflects the new patch's anchor-tone values. No leftover offset is
   applied; nothing audibly "snaps" or resets. Switch back — original (un-edited) patch
   values reappear (edits to a temp patch are not persisted across reload unless saved).

4. **macro_mode=relative restores old behavior.** Set param `macro_mode` = `relative`.
   Macro knobs now read 0 (offset) and adding an offset shifts all 4 tones non-destructively
   via SysEx without writing NVRAM; the offset clears on program change. Set
   `macro_mode` = `absolute` to return to the default. Verify `get_param("macro_mode")`
   reports the current mode string. This A/B switch requires no rebuild and no UI change.

5. **save_to_slot captures edits.** In absolute mode, edit a macro (e.g. raise Cutoff), then
   `do_save_to_slot` to a user slot. Reload that slot — the edited tone values persist.

6. **State round-trip.** Save chain state (autosave), reload. `macro_mode` and the edited
   working patch are restored from the state JSON (version 2). A v1 state with no
   `macro_mode` field loads with the default (absolute).

---

## Phase B — Tone editing (nested hierarchy + value formatting + save)

**Renderer note (READ FIRST).** When the module runs in a Signal Chain slot, the *platform's*
`shadow_ui.js` renders these menus — NOT the module's own `src/ui.js`. That renderer keys every
param entry as `synth:<key>`, has **no** `tone_section`/`tone_prefix` support, and does **not**
carry a `child_prefix` context into a *navigated* sub-level. Therefore per-tone editing uses
**explicit per-tone levels** (`tone1`..`tone4`, and `tone1_wave`..`tone4_fx`) whose param entries
are **fully-qualified** `nvram_tone_<N>_<param>` keys (N=0..3), each registered 1:1 in
`chain_params`. This is the dx7 "operators written out explicitly" pattern. (The previous build
invented `tone_section`/`tone_prefix` + a single shared selected-tone index, which the chain
renderer ignored — hence no values, no editing, dead knobs, broken Back.)

Navigation: from a patch, click the jog to open the patch edit menu (`patch_main`), choose
**Edit Tones** (`tone_selector`) → **Tone N** → section. Max depth: root → Tone N → section.

### 1. Tone selection

1. **Tone menu.** In **Edit Tones** (`tone_selector`), the four entries **Tone 1–4** each
   navigate into a distinct per-tone level. There is no "active tone tab" / Left-Right tone
   switching — each tone is its own explicit branch.
2. **Tone Switch.** Each **Tone N** menu has a `Tone Switch` row showing **Off/On** (not 0/1);
   click to toggle and confirm the tone mutes/unmutes audibly.
3. **Tones are independent.** Open **Tone 3 → Filter**, edit Cutoff — only Tone 3's filter
   changes (Tones 1/2/4 unaffected). Back out, open **Tone 1 → Filter** — it edits Tone 1.

### 2. Knob row follows the tone (every per-tone level)

4. **Knob row is live on each Tone N menu and every section under it.** Knobs map to (tone N,
   0-indexed): Cut(`nvram_tone_N_cutofffrequency`) Res(`…_resonance`) Atk(`…_tvaenvtime1`)
   Dcy(`…_tvaenvtime2`) Sus(`…_tvaenvlevel3`) Rel(`…_tvaenvtime4`) Lvl(`…_level`) Pan(`…_pan`).
   Inside Tone N and its Wave/Pitch/Filter/Amp/LFO/FX sublevels, turning each knob changes that
   tone's param audibly. Open a different Tone — the knobs now drive that tone.

### 3. Sections edit audibly

5. **Wave.** wavegroup (enum INT-A/INT-B/EXP-A/EXP-B), wavenumber, fxmswitch (Off/On),
   fxmdepth, toneswitch. Changing wavenumber changes the sampled waveform.
6. **Pitch.** coarse, fine, keyfollow, random pitch, pitch env depth + T1–4/L1–4.
   Coarse ±N audibly transposes the tone.
7. **Filter.** mode (Off/LPF/HPF enum), cutoff, resonance, keyfollow, reso mode, TVF env
   depth + velo + T1–4/L1–4. LPF cutoff sweep is audible; HPF thins the low end.
8. **Amp.** level, pan, level/pan keyfollow, velo sense/curve, TVA env T1–4/L1–4, tone delay
   mode (enum Normal/Hold/Play-Mate/Clock)/time. Attack/Release changes shape the envelope.
9. **LFO 1 / LFO 2.** form (TRI/SIN/SAW/SQU/RND1/RND2 enum), rate, delay, fade, offset (LFO1),
   sync (LFO1 only), pitch/TVF/TVA depths. Raising LFO1 pitch depth introduces vibrato.
10. **FX Sends.** drylevel, reverbsendlevel, chorussendlevel — raising reverb send increases
    the tone's reverb tail.

### 4. Enum / value display

11. **Enums show names, not numbers.** Verify: filter mode reads **Off/LPF/HPF**; LFO form
    reads **TRI/SIN/SAW/SQU/RND1/RND2**; wave group reads **INT-A/…**; on/off switches read
    **Off/On**; delay mode reads its names. None display a bare integer. (The DSP returns the
    numeric index; the platform renderer maps it to the label via the `options` array.)
12. **Signed values show their sign.** Pitch coarse/fine, env depths, LFO depths, and velo
    senses display the correctly-signed integer (e.g. `-7`, `0`, `12`), because the DSP's
    `get_param` returns a signed value for these keys.
    **Known cosmetic limitation:** the chain renderer does NOT honor a bare `display` metadata
    field (it only reads `display_unit`/`display_format`/`display_value_type`). So pan renders
    as a signed integer (`-64`..`63`) rather than **C / Lnn / Rnn**, and signed params show no
    explicit leading `+`. Values are correct and editable; only the cosmetic L/R/± skin is
    absent in chain context. (Module-standalone `src/ui.js` can still apply these.)

### 5. Save to slot + reload round-trip

13. **Save flow.** From the patch edit menu choose **Save to Slot** (`save_slot`). The list
    shows all 64 user slots with their stored names (or `(empty)`); current names render as
    `NN: NAME`. Select a slot and confirm — the working patch (including all tone edits and
    knob-row changes) is written via `do_save_to_slot`; the UI returns to the patch screen
    (`navigate_to: "patch"`). The `save_slot`, `expansions`, and `load_expansion` levels use the
    renderer's supported `items_param`/`select_param`/`navigate_to` dynamic-list mechanism.
14. **Reload round-trip.** Load the slot just saved (User patch load path). Re-open the edited
    section — the edited values persist. Edit a tone, save to a different slot, reload the
    *original* preset — the original (un-edited) values reappear (edits only persist in saved
    slots).

### 6. Back navigation

15. **Back exits every sublevel.** From `tone3_filter`, Back returns to `tone3`; Back again to
    `tone_selector`; Back again to `patch_main`; Back again to the patch browser. The renderer
    discovers parents by reverse-scanning levels for one whose `params[].level` points at the
    current level — every per-tone level is reachable from exactly one parent, so Back is
    unambiguous at every depth. (The old build's `tone_section` levels were unreachable-by-Back
    because no parent referenced them as a `level` nav target with a live child context.)

### 7. Screen-reader announcements

16. **Announcements fire.** Entering a level logs `[view] <label>`; selecting a param to edit
    logs `[item] <name>: <value>` with the formatted (enum/signed) value. Confirm in the
    console that level changes and edit-entry both announce. (No dedicated TTS channel exists
    yet; these are the named hook points for a future screen-reader layer.)
