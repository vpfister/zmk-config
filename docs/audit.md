# Audit of the current code state

Date: 2026-09-17
Branch: `vibe/mac-cleanup-audit-96531517`
Base: `origin/main` @ `ab7187b` ("flake: add git to devshell packages")

This document audits the fork's state relative to its upstream
[`urob/zmk-config`](https://github.com/urob/zmk-config) (fetched @ `a7232a1`,
"Harden CI"). The fork is a clean fork: `origin/main` is a strict ancestor of
upstream `main`, so no local commits exist that need preserving.

## 1. Repository layout

```
├── build.yaml                  # GitHub Actions build matrix (5 targets)
├── config/
│   ├── base.keymap             # shared 34-key keymap (layers, HRMs, behaviors)
│   ├── combos.dtsi             # all combos
│   ├── leader.dtsi             # leader-key sequences (umlauts, Greek, system)
│   ├── mouse.dtsi              # pointing-device tuning + aliases
│   ├── corneish_zen.{keymap,conf}  # 36-key split (the user's chocofi board)
│   ├── planck_rev6.{keymap,conf}   # 48-key wired ortholinear
│   ├── glove80.{keymap,conf}       # 80-key split (Moergo Glove80)
│   └── west.yml                # west manifest (ZMK + Zephyr + 5 modules)
├── draw/                       # keymap-drawer config + generated SVG/PNG
├── nix/                        # keymap-drawer + tree-sitter-devicetree packages
├── flake.nix / flake.lock      # nix devshell (west, zephyr-sdk, cmake, ...)
├── Justfile                    # build/draw/test/init recipes
└── .github/workflows/          # build.yml, build-nix.yml, test-build-env.yml
```

## 2. Naming issues (user complaint: "naming is a bit hard to understand")

- **Board files use legacy board names.** `corneish_zen.keymap` builds boards
  `corneish_zen_v2_left/right`. Upstream renamed the build targets to the
  hardware-manifest v2 names (`corneish_zen_left@2.0.0//zmk`), but kept the
  file name. The *file* name should ideally be `corneish_zen_v2.keymap` or
  the target should stay `corneish_zen_v2_*` — currently the file name and
  the board names disagree, which is confusing.
- **Layer indices are bare numbers.** `DEF 0`, `NAV 1`, `FN 2`, `NUM 3`,
  `SYS 4`, `MOUSE 5` are defined in `base.keymap` and used across
  `combos.dtsi`, `mouse.dtsi`, and `leader.dtsi`. Reading `combos.dtsi`
  requires jumping back to `base.keymap` to decode them.
- **Key labels are positional but inverted.** `LT0` is the *innermost* top-row
  key and `LT4` the outermost (see `zmk-helpers/key-labels/36.h`). This is
  documented upstream but non-obvious when reading the keymap grid.
- **Behavior instance names vs. combos.** E.g. `comma_morph`, `dot_morph`,
  `qexcl`, `lpar_lt`, `rpar_gt` are mod-morphs, while `esc`, `mouse`, `tab`,
  `ldr`, `cut`, ... in `combos.dtsi` are combos. The two namespaces are
  visually similar but semantically different.
- **`XXX` / `___`** are defined (`&none` / `&trans`) but only `___` is used;
  `XXX` is dead weight.

## 3. Legacy items and dead code

- **`#define XXX &none`** in `base.keymap` — unused. Remove.
- **`// Misc aliases. [TODO: clean up]`** block in `base.keymap` — the TODO has
  been there since the initial port. Aliases like `DSK_PREV`, `DSK_NEXT`,
  `PIN_WIN`, `PIN_APP`, `DSK_MGR`, `VOL_DOWN` are Linux/GNOME-flavored
  (`LG(LC(...))` = GUI+Ctrl). They are unused on the chocofi's layers? —
  actually they *are* used on the Fn layer. But they are Linux-specific and
  the user now works on macOS; they should be reworked for macOS (Cmd-based)
  or moved to a dedicated `#ifdef`.
- **`&caps_word { /delete-property/ ignore-modifiers; };` with comment
  "requires PR #1451. [TODO: rebase]"** — upstream ZMK PR
  [#1451](https://github.com/zmkfirmware/zmk/pull/1451) was **closed unmerged**
  (superseded by smart-layers work). The property deletion is therefore still
  required; the TODO is stale and misleading. Upstream kept the workaround,
  so we keep it too — but the TODO should be reworded.
- **`&bootloader` on the Sys layer** — known upstream limitation
  ([zmkfirmware/zmk#1086](https://github.com/zmkfirmware/zmk/issues/1086)):
  does not work on STM32 boards (Planck). Still open upstream. Keep, but
  document.
- **Tap-only combos workaround** (`ZMK_COMBO_8` / `hm_combo_*` hack) — still
  needed; upstream issue
  [zmkfirmware/zmk#544](https://github.com/zmkfirmware/zmk/issues/544) is
  still open. Keep.
- **Planck Rev6 and Glove80 configs** — the user only builds the chocofi
  (Corne-ish Zen, 36 keys). These are inherited from upstream and are not
  used. Keep for now (they document the multi-board pattern) but they are
  candidates for pruning. See plan.
- **`readme.md` (lowercase)** — upstream renamed it to `README.md` and
  rewrote it substantially. GitHub renders either, but the rename avoids
  confusion and matches upstream.
- **`build-nix.yml` references `urob/zmk-actions`** — works, but for a fork it
  may be preferable to use `zmkfirmware/zmk/.github/workflows/build-user-config.yml`
  (as `build.yml` does) or a pinned ref. Upstream moved to `@main`; consider
  pinning a tag for reproducibility.

## 4. Outdated dependencies (vs upstream `main` @ a7232a1)

| Component | Fork (pinned) | Upstream (pinned) | Notes |
|---|---|---|---|
| ZMK | `v0.2` (via urob fork, main @ 2025-03) | `641514a` main (2026-08-31) | Major bump; HWMv2 board names |
| Zephyr | `v3.5.0+zmk-fixes` | `v4.1.0+zmk-fixes` (2026-08-18) | Required by ZMK main |
| zmk-helpers | `v0.2` tag | `95edb8f` main (2026-08-31) | KEYS_L/KEYS_R/THUMBS now in key-labels headers |
| zmk-adaptive-key | `v0.2` | `e5b335a` main | |
| zmk-auto-layer | `v0.2` | `62c2019` main | |
| zmk-leader-key | `v0.2` | `e894d9a` main | |
| zmk-tri-state | `v0.2` | `0c6eaf5` main | |
| **zmk-unicode** | **absent** | `5f458b6` main | **New module — key for macOS support** |
| zephyr-nix (flake) | `urob/zephyr-nix` | `nix-community/zephyr-nix` | urob's fork is unmaintained for Zephyr 4.x |
| keymap-drawer | 0.21.0 | 0.23.0 | |
| native test board | `native_posix_64` | `native_sim//zmk_test_mock` | Required by ZMK main |

Key upstream keymap changes (port candidates):

1. **Switch to the `zmk-unicode` module** (upstream commit `2696607`):
   - `base.keymap`: add `#include <behaviors/unicode.dtsi>`, drop
     `#include "zmk-helpers/unicode-chars/greek.dtsi"` and `german.dtsi`.
   - `leader.dtsi`: replace `&de_ae` etc. with `&uc UC_DE_AE` etc.
   - `west.yml`: add `zmk-unicode` project.
   - The module supports **macOS (Unicode Hex Input via Option key),
     Linux (IBus), Windows (WinCompose / HexNumpad), Emacs**, switchable at
     runtime via `&uc UC_SET_*` and configurable via `&uc { default-mode = ...; }`.
     This is the single most valuable upstream change for the user's macOS
     request.
2. **Remove local `KEYS_L`/`KEYS_R`/`THUMBS` defines** (upstream `395f46b`):
   `zmk-helpers` now ships them in every key-labels header (since 2026-03-04,
   PR #100). Requires bumping the module past that commit.
3. **HWMv2 board names in `build.yaml`** (upstream `6b37b22`):
   `planck@6.0.0//zmk`, `corneish_zen_left@2.0.0//zmk`,
   `corneish_zen_right@2.0.0//zmk`, `glove80_lh/nrf52840`, `glove80_rh/nrf52840`.
4. **Close-window key on Nav layer** (upstream `8e8ba36`): `&kp LA(F4)` on
   Nav top-left — macOS/Windows/Linux all support Cmd/Ctrl+Alt+F4 or
   Alt+F4; useful cross-platform.
5. **Justfile overhaul**: renamed recipes (`update`→`sync`, `upgrade-sdk`→
   `bump-nix`, `clean-all`/`clean-nix` removed, new `flash`, `format`,
   `bump-west`), `_parse_combos` removed (combo config now upstream defaults),
   `test` uses `native_sim//zmk_test_mock`, `_check_yq_version` guard,
   board names with `/` are sanitized in artifact names.
6. **Flake**: Zephyr 4.1, `nix-community/zephyr-nix`, `pin-west` for manifest
   locking, `dts-format`/`dts-linter` packaged, `gcc` added, `PYTHONPATH`
   set for zephyr pythonEnv, Linux-only `LD_LIBRARY_PATH` for libatomic.
7. **CI**: `test-build-env.yml` now tests on macOS (`macos-15-intel`,
   `macos-15`) — the fork already had `macos-13`/`macos-15`; upstream moved
   to `macos-15-intel` (x86_64-darwin). `build-nix.yml` triggers on PRs too.
   New `bump-west.yml` workflow.
8. **Docs**: upstream split the readme into `README.md` (keymap + build) +
   `docs/build-env.md` (nix setup) + `AGENTS.md` (customization guide).

## 5. macOS gaps in the current fork

- **Unicode input**: the fork uses `zmk-helpers/unicode-chars/*.dtsi`, which
  emit raw `ZMK_UNICODE_PAIR` sequences using the **Linux IBus**
  (`Ctrl+Shift+U ... Space`) input method only. On macOS this does nothing
  unless the host is configured (and even then IBus is not available). The
  `zmk-unicode` module adds a proper macOS mode (Option-hex via "Unicode Hex
  Input" input source) plus runtime switching.
- **Layer shortcuts**: `DSK_PREV`/`DSK_NEXT`/`PIN_WIN`/`PIN_APP`/`DSK_MGR`
  use `LG(...)` (GUI = Cmd on macOS, Super on Linux). `LG(LC(LEFT))` is
  "Move left a space" on macOS — works. `LG(LC(LS(Q)))` (Pin window) and
  `LG(LC(LS(A)))` (Pin app) are GNOME-specific and do nothing useful on macOS.
  `LA(GRAVE)` (Desktop manager / Mission Control on macOS via Alt+`?) —
  actually `LA(GRAVE)` is Option+Grave, not Mission Control (F3). Needs a
  macOS-appropriate binding (`LG(F3)` or `LG(UP)`).
- **Close window**: Nav layer has no close-window key; upstream added
  `&kp LA(F4)` (Alt+F4 on Linux/Windows; on macOS Option+F4 is not standard
  — `LC(W)` is Cmd+W). Worth adding `&kp LC(W)` or keeping `LA(F4)`.
- **Build environment**: the nix devshell already supports
  `x86_64-darwin`/`aarch64-darwin` and CI tests `macos-13`/`macos-15`, so
  local builds on a Mac should work today. Upstream additionally fixed the
  devshell on Darwin (`694cf25`) and moved `zephyr-nix` to the
  nix-community repo for Zephyr 4.x support.
- **Swapper (Alt-Tab)**: on macOS the equivalent is Cmd-Tab, not Alt-Tab.
  The `&swapper` tri-state sends `LALT`+`TAB`. For macOS, `LGUI`+`TAB` is
  the app switcher. Consider making the swapper mod configurable or adding a
  macOS variant.

## 6. Build/CI state

- `build.yml` delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`
  — unpinned, follows upstream ZMK main. Works, but pinning a tag is safer.
- `build-nix.yml` delegates to `urob/zmk-actions/...@main` — same concern.
- `test-build-env.yml` runs `just init`, `just build planck`, `just draw` on
  4 systems including two macOS runners. Good coverage; the Planck target
  will break if board names change to HWMv2 style unless `build.yaml` is
  updated in the same commit (it is, upstream `6b37b22`).
- `.envrc` uses `use flake` + `watch_file nix/*`. Fine.
- `Justfile` `_parse_combos` rewrites `CONFIG_ZMK_COMBO_MAX_*` in all `.conf`
  files on every build — mutates tracked files as a side effect. Upstream
  removed this; ZMK's defaults now suffice (max 6 combos/key, 3 keys/combo
  are upstream defaults since the combo rework). Porting the removal avoids
  dirty working trees.

## 7. Summary of recommended actions

See [plan.md](plan.md) for the concrete, ordered task list.
