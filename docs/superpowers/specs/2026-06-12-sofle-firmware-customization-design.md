# Sofle Firmware Customization — Design

**Date:** 2026-06-12
**Repo:** `~/Dev/zmk-config` (fork of `MechboardsLTD/zmk-config`), branch `refresh_mips`
**Goal:** One clean source-first build that fixes battery drain, makes the rotary encoders layer-aware, and reworks the displays — replacing the runtime-only ZMK Studio layout with a version-controlled keymap.

## Motivation

The wireless Sofle (nice!nano v2, nice!view displays) was draining ~50%/day on both halves. Root cause found by reading `config/sofle.conf`: **there is no `CONFIG_ZMK_SLEEP` line, so deep sleep was never enabled** (defaults off). While fixing that we also bake in the layout (currently only in ZMK Studio's on-device settings) and do the long-pending encoder + display work as one build.

## Source-of-truth decision

**Source-first.** ZMK Studio has no export-to-`.keymap` feature ([zmk-studio#124](https://github.com/zmkfirmware/zmk-studio/issues/124)), and since this build edits the keymap anyway (encoders + ADJUST-layer cleanup), relying on the Studio snapshot surviving a reflash is fragile. So the `.keymap` becomes the single source of truth; the build ends with a `settings_reset` cutover.

## Components

### 1. Battery / power — `config/sofle.conf`
- **`CONFIG_ZMK_SLEEP=y`** + `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT_MS=900000` — the actual drain fix (deep sleep after 15 min idle). nice!view is persistent, so the frozen last frame in deep sleep is expected, not a fault.
- **`CONFIG_ZMK_RGB_UNDERGLOW=n`** — no underglow LEDs are installed; also removes the footgun where the underglow toggle cut ext-power to both displays (shared rail).
- BT TX power left at `+8dBm` — helps the split-link robustness; sleep is the real battery lever, not TX power.
- *Status: already drafted in the working tree.*

### 2. Keymap — `config/sofle.keymap`
- **Re-author the keymap in source from documented intent (approach B), not screenshot extraction.** ZMK Studio's overlay shows base keycodes but hides hold-behaviors (mod-tap, layer-tap, held mods), so screenshots miss exactly the interesting keys. Since we're cutting over to source-first, the on-device Studio state isn't sacred — the target is the layout wanted *going forward*. Draft from the memory notes + this spec (z↔y physical swap, Caps→tap-Esc/hold-MEH, brackets on the LOWER layer right hand, thumbs L=Enter/R=Space + LOWER/RAISE on thumbs, layer-tap backspace on the right layer key), review together layer-by-layer, flash, and refine on hardware. Studio is consulted only to settle specific per-key questions. User's screenshots serve as a helpful cross-check basis.
- **ADJUST-layer cleanup (forced by underglow off):** the `&rgb_ug RGB_*` and `&ext_power EP_TOG` bindings will no longer compile; replace with `&none` (or repurpose the freed space later).
- **Encoders — per-layer `sensor-bindings`.** Ergonomic rule: a layer's knob action goes on the hand *opposite* the thumb holding that layer (so the free hand turns it). ADJUST holds both thumbs → no usable knob.

  | Layer | Left knob | Right knob |
  |---|---|---|
  | Base | Horizontal scroll | Vertical scroll |
  | RAISE (held by R thumb) | Zoom (`Cmd +/−`) | — |
  | LOWER (held by L thumb) | — | Undo/Redo (`Cmd+Z` / `Cmd+Shift+Z`) |
  | ADJUST | — | — |

  Knob clicks: left = Mute (existing), right = unused — left as-is, not redesigned. Undo/Redo is GUI-only (no-op in terminal Neovim, where native `u`/`Ctrl+R` already cover it) — accepted trade for a seldom-used slot.

### 3. Display — fork `MechboardsLTD/zmk-module` (`nv_gem`), repoint `config/west.yml`
The module splits by role: left = central (`screen.c`: battery, output/BT, layer, WPM), right = peripheral (`screen_peripheral.c`: the `crystal.c` animation).
- **Remove the WPM gauge + chart** from the central screen (frees the vertical space the left-half logo needs).
- **Keep** battery + output/BT signal + layer.
- **Star Wars split (approach B):** custom 1-bit LVGL bitmaps in the module's `assets/`.
  - **Left half → Rebel Alliance starbird**
  - **Right half → Galactic Empire cog** (replaces the crystal animation)
- Animation already off (`CONFIG_NICE_VIEW_GEM_ANIMATION=n`) → clean static emblems.

### 4. Build & flash
- Push fork → GitHub Actions builds the `.uf2`s (matrix in `build.yaml`).
- Flash `settings_reset`, then flash `sofle_left` + `sofle_right` — the source-first cutover. (Do **not** preserve old Studio settings; source is now authoritative.)
- Validate: after a full charge, leave both halves idle past 15 min → host shows BT disconnect (sleep engaged) → drain should flatten. Measure from full charge (LiPo % is voltage-based + non-linear; first-cycle readings are unreliable).

## Open dependency
**Keymap co-authoring session (component 2)** — a focused layer-by-layer pass done together, drafted from documented intent (approach B). The user's per-layer ZMK Studio screenshots serve as a cross-check basis. Everything else (battery conf, display fork + art) can proceed in parallel — display work is starting now.

## Out of scope (YAGNI)
- Knob-click redesign.
- Re-enabling underglow / any RGB.
- Clever Neovim-aware undo on the encoder.
- Adjust-layer encoder bindings (physically unreachable).
