# Color Timer Clock Desklet - Development Guide

## Overview

The Color Timer Clock desklet shows up to three cards (Clock, Timer, Chronometer) in one row. Each card's background color is evaluated every second from a schedule of (time, color) stops and applied through one cached inline style, so the CSS transition-duration alone decides between a 1000 ms ramp and a 0 ms step.

**UUID**: `cinnamon-color-timer-clock-desklet@curbsoftware`

**Type**: Cinnamon Desklet

---

## Features

- Clock card: time-of-day color schedule wrapping over midnight, optional IANA timezone
- Timer card: countdown by time-remaining schedule; `stopped|paused|running|expired` state machine with wall-clock `endMs`, persisted across restarts
- Chronometer card: elapsed-time schedule; `accumMs` + `startMs` pause/resume, persisted across restarts
- On-card St.Button controls (play/pause, reset, timer ±60 s while stopped) — no PopupMenu, so none of the grab/teardown/deferral bug class; every control carries a tooltip
- Optional chronometer hundredths display: a 50 ms text-only tick armed only while the chronometer runs with the setting on
- Adaptive text/border colors by measured WCAG contrast ratio (translucent colors are composited over the dark card backing first; mid-tones that fail AA with either softened candidate fall back to the solid extreme)
- "Reset all schedules to defaults" settings button

---

## Project Structure

```
cinnamon-color-timer-clock-desklet@curbsoftware/
├── desklet.js              # CardWidget, grid, tick, timer/chrono state machines
├── cardActions.js          # Pure color engine + card-layout maths
├── metadata.json           # UUID, name
├── settings-schema.json    # Cards, schedules, display, reset button, state stores
├── stylesheet.css          # Card chrome (ctc- namespace)
├── icon.png                # 128x128
├── screenshot.png
├── po/                     # Translation template
├── README.md
└── DEVELOPMENT.md
```

Cinnamon cannot import across xlets. `computeCardInnerSize` / `computeFittedFontSizes` are copied verbatim from the World Clock desklet's `clockActions.js` (tile renamed to card).

---

## Development Setup

```bash
./dev-tools/install-extensions.sh -m symlink -n '*color-timer*'
```

The install directory name must match the UUID: `~/.local/share/cinnamon/desklets/cinnamon-color-timer-clock-desklet@curbsoftware/`.

Reload without restarting Cinnamon:

```bash
gdbus call --session --dest org.Cinnamon --object-path /org/Cinnamon \
  --method org.Cinnamon.ReloadXlet 'cinnamon-color-timer-clock-desklet@curbsoftware' DESKLET
```

Logs:

```bash
tail -f ~/.xsession-errors | grep -i color-timer
```

---

## Architecture

```
Desklet.Desklet
    └── MyDesklet
            ├── _rebuildCards()         St.Table 1xN of CardWidgets
            ├── _updateAll()            1s tick: expiry, evaluate(), paint
            ├── _timerControl()         stopped|paused|running|expired
            ├── _chronoControl()        stopped|paused|running
            └── _cleanup()              destroy + on_desklet_removed

CardWidget(kind)
    └── St.BoxLayout.ctc-card
            ├── title label
            ├── value label
            ├── sub label
            └── controls pill (timer/chrono only): play/pause, reset, ±60 s

cardActions.js     parseColor, normalizeSchedule, evaluate, luma/contrast,
                   computeCardInnerSize / computeFittedFontSizes
```

Modules are loaded lazily:

```javascript
const dir = imports.ui.deskletManager.desklets[uuid];
CardActions = dir.cardActions;
```

### Per-second tick

`_updateAll()` computes each visible card's position (clock: seconds of day in the card timezone; timer: remaining ms; chrono: elapsed ms), calls `evaluate(stops, pos, { wrap, smooth })` and hands the rgba to `CardWidget.update()`. A color key (rgba + smooth + margin) skips no-op `set_style` calls; the CSS transition bridges the 1 s sampling. Font fitting never runs from the tick.

### Timer state machine

`endMs` is wall-clock while running, so restarts during a run resolve correctly: `_restoreState()` maps a running timer whose `endMs` has passed to `expired`. State is persisted only on transitions (`_persistTimer`), never per tick. Pausing after `endMs` transitions straight to `expired` — `_updateAll` only expires a running timer.

### Schedules

`normalizeSchedule(rows, maxT)` parses rows (clock `hour`/`minute`, timer `remaining`, chrono `elapsed`, or bare `t`), clamps, drops unparsable rows (counted in the log line), sorts and keeps the later row on equal times. `resetSchedules()` writes the defaults with `setValue` and refreshes `_scheduleRows` by hand: in-process `setValue` does not re-fire bind callbacks (the bound properties themselves are live getters and do update).

---

## Settings

| Key | Type | Default | Notes |
|-----|------|---------|-------|
| `show-clock` / `show-timer` / `show-chronometer` | checkbox | `true` | Visible cards |
| `clock-timezone` | entry | `""` | Empty = local; invalid logs once and falls back |
| `clock-schedule` | list | 06:00/12:00/18:00 ramp | `hour`, `minute`, `color` columns |
| `timer-minutes` / `timer-seconds` | spinbutton | `5` / `0` | Default duration |
| `timer-schedule` | list | 60/30/0 s green-yellow-red | `remaining`, `color` columns |
| `chrono-schedule` | list | 0/1800 s teal-orange | `elapsed`, `color` columns |
| `*-smooth` | checkbox | `true` | Interpolate vs step |
| `time-format` / `date-format` | entry | `%H:%M:%S` / `%A, %e %B` | strftime via `toLocaleFormat` |
| `time-size` / `date-size` / `label-size` | spinbutton | `34` / `13` / `11` | pt caps |
| `card-spacing` | spinbutton | `6` | Card margin px |
| `width` / `height` | spinbutton | `840` / `260` | Desklet bounds |
| `reset-schedules` | button | — | Callback `resetSchedules` |
| `timer-state` / `chrono-state` | generic | stopped | Persisted state stores |

Colors accept `#rgb`, `#rgba`, `#rrggbb`, `#rrggbbaa` (either case) and 13 CSS names; anything else drops the row with a log line.

---

## Color engine API (`cardActions.js`)

All functions are pure (no St/Clutter):

- `parseColor(str)` → `{ ok, rgba }`
- `rgbaToCss` / `rgbaToKey` — style string / canonical change-detection key
- `normalizeSchedule(rows, maxT)` → `{ stops, dropped }`
- `evaluate(stops, t, { wrap, smooth })` → rgba or `null` (no stops)
- `lerpRgba`, `luma`, `contrastColors(rgba)` → `{ fg, border }`
- `computeCardInnerSize` / `computeFittedFontSizes` — copied from clockActions.js

---

## Testing

```bash
gjs dev-tools/test-card-actions.js          # pure engine + layout maths
python3 dev-tools/live-test-color-desklet.py  # live driver (needs a session)
```

Headless coverage: parse variants, normalize (sort/clamp/drop/merge), evaluate (wrap both directions, step-at-stop, smooth midpoint, clamps, empty), luma/contrast, fit helpers. Widget behaviour needs a live Cinnamon session.

### Manual check

1. Install with the symlink command above and reload the desklet.
2. Smooth ramp visibly animates over a 60 s timer; step mode jumps exactly at stops.
3. Red→blue passes purple; the 18:00→06:00 clock wrap ramps through the night.
4. Yellow card → dark text; a malformed color row logs a fallback, the card never blanks.
5. Timer survives ReloadXlet to the correct expiry; expiry during downtime shows expired on load; chrono accumulates across pause/resume/reload.
6. show-* toggles rebuild cleanly; width 200–2000 sweep: no overflow, equal cards.

---

## Gotchas

- `Desklet.destroy()` emits `destroy` immediately but defers `on_desklet_removed` by a 500 ms fade. Hook both; `_cleanup()` is idempotent.
- Never `Mainloop.idle_add` for user-visible actions; use `timeout_add(0)`.
- St ignores `transition-property`; a `transition-duration` alone crossfades the whole node paint. The card's inline style owns the duration (1000 ms smooth / 0 ms step) — do not move it into the stylesheet.
- The card's entire inline style (margin + colors + duration) is one string; a separate `set_style` anywhere on the actor would clobber the rest.
- Invalid IANA timezone ids log once per distinct bad value (the check runs every second) and fall back to local time.
- `toLocaleFormat` is a GJS `Date` extension (strftime).
