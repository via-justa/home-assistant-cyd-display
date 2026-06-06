# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project overview

ESPHome + LVGL smart-home dashboard for a Cheap Yellow Display (ESP32-2432S028,
aka "CYD"): a 320x240 ILI9341 display with an XPT2046 resistive touchscreen,
driven by an ESP32 over two SPI buses. The UI shows 2x2 control tiles, a media
row, and a header (room name / weather / clock). It talks to Home Assistant over
the native API.

## File layout

The config is split into package fragments included from `dashboard.yaml`:

- `dashboard.yaml` — entry point: system + hardware (esp32, wifi, spi, display,
  touchscreen, time, globals) and the `packages:` includes. **Always validate /
  build from this file.**
- `config.yaml` — user substitutions (entities, colors, tile config, flags).
  Gitignored-style local copy of `config.yaml.example`.
- `config.yaml.example` — template for `config.yaml`; When adding a substitution, add the new key (with its aligned `#` comment) to `config.yaml.example` first, then add the same key to the local `config.yaml` if it is present on disk. Never omit the entry from `config.yaml.example` — that file is the canonical template committed to the repo.
- `secrets.yaml` — local credentials (gitignored). Template: `secrets.yaml.example`.
- `sensors.yaml` — `sensor`, `binary_sensor`, `text_sensor`.
- `lvgl.yaml` — `font`, `image`, and the full `lvgl` UI (styles, pages, widgets).
- `scripts.yaml` — all automation / UI logic scripts (e.g. `ui_refresh`).
- `debug_tools/` — optional debug package, gated by a flag (see below).
- `components/display_capture/` — local external component for on-device screenshots.
- `fonts/` — font binaries (local Heebo TTFs + Material Design Icons webfont).
- `images/` — background image and README screenshots.

## Commands

Use the project's virtualenv:

```bash
source ~/esphome-venv/bin/activate
esphome config dashboard.yaml      # validate
esphome run dashboard.yaml         # build + flash / OTA
```

- Validate **only** via `dashboard.yaml` — the other YAML files are package
  fragments and won't validate standalone.
- If `esphome config dashboard.yaml` exits with an error, stop, show the full
  error output to the user, and do not proceed to `esphome run`. Do not attempt
  to auto-fix validation errors unless the user explicitly asks.
- If `config.yaml` is absent, copy `config.yaml.example` to `config.yaml` before
  running any `esphome` command. Do not create it from scratch; always base it
  on the example file.
- `esphome run` does NOT exit after flashing — it streams device logs and stays
  attached until interrupted (Ctrl-C). Don't wait for it to return on its own.
- `-s KEY VAL` does NOT override substitutions loaded via `!include config.yaml`;
  to test a different flag value, edit `config.yaml` (and revert).

## Conventions

1. Declare every substitution value as a quoted string, even booleans and
   numbers (e.g. `TIME_24H: "true"`, `AUTO_DIM_TIMEOUT: "10"`).
2. Inside C++ lambdas, always wrap substitution expansions in `std::string()`
   before comparing (e.g. `std::string("${AUTO_DIM}") != "true"`) — never
   compare the raw expansion directly.
- When adding a new substitution, add it to BOTH `config.yaml` and
  `config.yaml.example` with an aligned trailing `#` comment describing it.
- Keep secrets out of git — reference them with `!secret`, never inline.
- Material Design icon glyphs are declared once in the `MDI_GLYPH_*` /
  `MDI_MEDIA_*` substitutions and reused; don't duplicate a glyph.

## Conditional components (no native `if`)

ESPHome has no Jinja/conditionals. To enable/disable a block by flag, select the
include path with a substitution:

```yaml
packages:
  debug_tools_pack: !include debug_tools/debug_tools.${DEBUG_TOOLS}.yaml
```

- `debug_tools/debug_tools.true.yaml` holds the real config; the `.false.yaml`
  variant is an empty `{}` placeholder. `DEBUG_TOOLS` defaults to `"false"`.
- This `.true.yaml` / `.false.yaml` pattern is the only supported way to
  conditionally include a component. If you need to add a new optional feature,
  always follow this same pattern — do not use Jinja conditionals,
  `!include_dir_*`, or any other mechanism.
- Package merge semantics: dicts merge recursively, **lists concatenate**. So a
  package can append entries to an existing list (e.g. `lvgl.displays`) rather
  than redefining it.

## LVGL / fonts notes

- Hebrew/RTL text requires LVGL bidirectional support, which ESPHome does not
  expose. It is enabled via a build flag in `dashboard.yaml`:
  `esphome: → platformio_options: → build_flags: ["-DLV_USE_BIDI=1"]`. Without
  it, Hebrew renders reversed and `base_dir: AUTO` is a no-op.
- Text fonts use local Heebo TTFs (weights: 400=Regular, 500=Medium, 700=Bold),
  which include the Hebrew glyph block. Icon fonts use the MDI webfont.
- Changing LVGL build flags / `lv_conf.h` triggers a clean rebuild (slower first
  build).

## Scope discipline

- Make only the change requested; Make only the change explicitly requested. Do not rename identifiers, reformat YAML, or reorganize file sections unless the request specifically asks for it. If you notice a separate problem, describe it in a comment rather than fixing it.
- `touchscreen.transform` must match `display.transform`; calibration values are
  device-specific — don't change them unless asked.
- Don't add or commit `secrets.yaml` or `config.yaml`.
