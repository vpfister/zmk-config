# Plan: macOS support, upstream ports, and cleanup

Date: 2026-09-17
Branch: `vibe/mac-cleanup-audit-96531517`
Based on the [audit](audit.md).

Guiding constraints:

- The user's keyboard is a **36-key chocofi split** built from the Corne v2
  template → the **Corne-ish Zen** target is the one that must keep working.
  Planck Rev6 and Glove80 configs are inherited from upstream and unused;
  they are kept working but are secondary.
- Changes land as **small, regular commits** on the branch.
- We **do not bump to ZMK main / Zephyr 4.1** in this pass: that is a large,
  separately testable step (HWMv2 board renames, `native_sim` test board,
  new flake inputs). It is documented as a follow-up. Everything done here
  stays on the currently pinned `v0.2` dependency set, which is known to
  build.
- The `zmk-unicode` module has **no v0.2-compatible release** (tags: v0.3,
  v0.3.0 only; it requires ZMK ≥ v0.3). Porting it therefore **requires**
  bumping the west manifest to ZMK main + Zephyr 4.1 + zmk-helpers main.
  Since macOS Unicode support is an explicit user goal, we do the full
  dependency bump, but as its own clearly-marked commit group so it can be
  reverted independently if the build proves problematic.

## Phase 0 — Docs (this commit)

- [x] `docs/audit.md` — audit of current state.
- [x] `docs/plan.md` — this file.

## Phase 1 — Cleanup that is safe on the current pins (v0.2)

1. **Remove dead `XXX` define** from `config/base.keymap`.
2. **Reword stale TODOs**:
   - `&caps_word { /delete-property/ ignore-modifiers; }` — PR #1451 was
     closed unmerged; the workaround is permanent. Update the comment.
   - `// Misc aliases. [TODO: clean up]` — resolve by restructuring (Phase 3).
3. **Rename `readme.md` → `README.md`** to match upstream and avoid
   case-confusion on case-insensitive macOS filesystems (HFS+/APFS are
   case-insensitive by default — a real source of confusion on a Mac).
4. **`.gitattributes`**: keep `*.keymap`/`*.dtsi` marked as C++.

## Phase 2 — Dependency bump to upstream-current pins (needed for zmk-unicode)

Ported from upstream commits `25f620d`, `37aceeb`, `1faf1bf`, `33c3925`,
`97b3688` and `5e8c2c8` (Zephyr bump):

1. `config/west.yml`:
   - ZMK → `641514a97db345f499dd50b0360e594270f008fe` (main, 2026-08-31)
   - Zephyr → `10ba6d0cb38bc3d258775d27982f707599320085` (v4.1.0+zmk-fixes)
   - Bump all 5 existing modules to their upstream-pinned revisions.
   - Add `zmk-unicode` @ `5f458b6f1fd8ca83f4aba793c2a64916522507e9`.
   - Add the pinned indirect dependencies (cmsis, hal_nordic, hal_rpi_pico,
     hal_stm32, lvgl, mbedtls, tinycrypt, zmk-studio-messages) — copied
     verbatim from upstream to avoid `west update` surprises.
2. `build.yaml`: HWMv2 board names
   (`corneish_zen_left@2.0.0//zmk`, `corneish_zen_right@2.0.0//zmk`,
   `planck@6.0.0//zmk`, `glove80_lh/nrf52840`, `glove80_rh/nrf52840`).
3. `flake.nix` + `flake.lock`: Zephyr `v4.1.0+zmk-fixes`,
   `zephyr-nix` → `nix-community/zephyr-nix`, add `pin-west`, `dts-format`,
   `gcc`, `PYTHONPATH`, Linux-only `LD_LIBRARY_PATH`. Port the
   `nix/dts-format.nix`, `nix/dts-linter.nix`, `nix/dts-lsp-server.nix`
   files. Bump `nix/keymap-drawer.nix` to 0.23.0.
4. `Justfile`: port upstream version (renamed recipes, `flash`, `format`,
   `_check_yq_version`, `native_sim//zmk_test_mock`, artifact-name
   sanitizing for `/` in board names, drop `_parse_combos`).
5. `base.keymap`: remove local `KEYS_L`/`KEYS_R`/`THUMBS` defines
   (now provided by zmk-helpers key-labels headers).
6. `.conf` files: drop the `CONFIG_ZMK_COMBO_MAX_*` lines (upstream removed
   the auto-tuning; defaults suffice).
7. CI: update `test-build-env.yml` matrix to `macos-15-intel`/`macos-15`
   and pin action SHAs as upstream does; `build-nix.yml` add `pull_request`
   trigger. Add `bump-west.yml`.

## Phase 3 — macOS support (keymap)

1. **Unicode via `zmk-unicode`** (port upstream `2696607` + macOS config):
   - `base.keymap`: `#include <behaviors/unicode.dtsi>`; remove
     `zmk-helpers/unicode-chars/{greek,german}.dtsi` includes.
   - `leader.dtsi`: `&de_ae` → `&uc UC_DE_AE`, etc. (all German + Greek
     sequences).
   - Set `&uc { default-mode = <UC_MODE_MACOS>; };` (user's main OS).
   - Add leader sequences or layer keys to switch input mode at runtime:
     e.g. `&uc UC_SET_MACOS`, `&uc UC_SET_LINUX`, `&uc UC_SET_WIN_COMPOSE`
     on the Sys layer so the same firmware works on all machines.
   - Document the macOS host setup (add "Unicode Hex Input" input source)
     in the README.
2. **Fix Linux/GNOME-specific aliases** on the Fn layer for macOS:
   - `DSK_MGR` (`LA(GRAVE)`) → Mission Control is `LG(F3)` / `LG(UP)` on
     macOS; keep `LA(GRAVE)` as a secondary or replace.
   - `PIN_WIN` / `PIN_APP` are GNOME-only; on macOS the equivalents don't
     exist as global shortcuts. Replace with macOS-useful keys or make them
     `&none` on a mac-focused layout.
   - Add `&kp LC(W)` (close window) — the universal macOS/Linux/Windows
     close shortcut; upstream added `LA(F4)` on Nav which is not idiomatic
     on macOS.
3. **Swapper mod**: the Alt-Tab swapper uses `LALT`+`TAB`. On macOS the
   app switcher is `LGUI`+`TAB`. Add a note / make the mod a `#define`
   (`SWAP_MOD`) so it can be flipped between `LALT` and `LGUI`.
4. **Nav layer close-window key**: add `&kp LC(W)` next to the upstream
   `&kp LA(F4)`.

## Phase 4 — Naming improvements

1. Rename `config/corneish_zen.keymap` → keep (matches upstream; renaming
   would diverge). Instead, add a comment header to each board file stating
   which `build.yaml` target(s) it serves, e.g.:
   `// Build targets: corneish_zen_left@2.0.0//zmk, corneish_zen_right@2.0.0//zmk`
2. Add a short "How to read this config" section to the README (or a
   `docs/keymap-guide.md`) explaining: layer defines, key labels (LT0 =
   innermost), the `ZMK_BASE_LAYER` adapter pattern, and where combos /
   leader sequences / mouse tuning live.
3. Layer defines: add a one-line comment block above `DEF/NAV/FN/NUM/SYS/MOUSE`
   listing them in one place (already there — keep).

## Phase 5 — Pruning (optional, user to confirm)

- The Planck Rev6 and Glove80 configs are unused by the user. Options:
  (a) keep them (upstream parity, they exercise the multi-board adapter
  pattern and CI builds them), or (b) remove them and slim `build.yaml` to
  the chocofi only. **Default: keep** — removing them diverges from upstream
  and loses the CI safety net; revisit if the user wants a minimal repo.
- `draw/keymap.png` is a stale generated artifact (upstream deleted it in
  favor of `draw/overview.svg`). Regenerate drawings with `just draw` and
  remove the PNG.

## Phase 6 — Docs refresh

- Port upstream's README restructure: rename to `README.md`, update the
  "builds against v0.2" line, add macOS Unicode setup instructions, add
  the layer-overview diagram (`draw/overview.svg`), and reference
  `docs/build-env.md`.
- Add `docs/build-env.md` (port from upstream).
- Add `AGENTS.md`? — optional; useful if the user wants agent-assisted
  keymap tweaks. Low priority; skip unless time permits.

## Verification plan

- No nix/west in this sandbox → full firmware builds cannot run here.
  Verification is by inspection + CI: push the branch and let
  `test-build-env.yml` (4 OSes incl. macOS) and `build-nix.yml` validate.
- Before each push: `git diff --stat` review, ensure no secrets, ensure
  generated files (`draw/base.yaml`, `.build/`, `firmware/`) are not staged
  (`.gitignore` covers them).
- `just draw` regeneration must be done in a nix env; if unavailable,
  leave SVG regeneration to CI/manual and note it.

## Out of scope / follow-ups

- Full rebase onto upstream `main` as a merge (would bring AGENTS.md,
  dts-linter, pre-commit, etc. — large diff, better done deliberately).
- ZMK Studio support (`zephyr-full` toolchain in build-nix.yml).
- Runtime OS detection (auto-switching Unicode mode per connected host) —
  not supported by zmk-unicode; would need a custom behavior.
