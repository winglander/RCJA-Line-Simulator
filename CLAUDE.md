# CLAUDE.md — Project Guide for Claude Code

## What This Project Is

A browser-based simulator for **RoboCup Junior Australia Rescue Line** (Riley Rover / Primary division). Students build LEGO Spike Prime robots and write Pybricks-style Python via a visual block editor. The simulator lets them test algorithms without a physical robot.

**Live:** https://winglander.github.io/RCJA-Line-Simulator/robocup-simulator.html
**Manual:** https://winglander.github.io/RCJA-Line-Simulator/docs/user-manual.html

## Architecture

### Single-file application
Everything lives in `robocup-simulator.html` (~2875 lines). No build tools, no frameworks, no npm. Vanilla JS + Canvas 2D. The only external dependencies are Google Fonts (CDN) and Blockly/Skulpt (CDN) for the code editor.

### Dev server
```bash
python -m http.server 8765
# Open http://localhost:8765/robocup-simulator.html
```
Configured in `.claude/launch.json`.

### Two coordinate systems
- **World coords** — millimetres (mm). All physics, robot position, sensor offsets.
- **Screen coords** — pixels. `SCALE() = _tilePixels / TILE_MM` converts mm → px.

### Key constants
```
TILE_MM = 594        # RCJA large tile = 594mm
MAP_COLS = 8         # arena grid width
MAP_ROWS = 6         # arena grid height
LINE_W_MM = 20       # black line width
HUB = {W:56, L:88}   # SPIKE Prime hub dimensions
LEVEL_HEIGHT = [0, 90, 180, 270]  # level elevations in mm
```

### Major code sections (in order in the file)
1. **CSS** (~130 lines) — dark theme, panel layout, sliders
2. **HTML structure** (~160 lines) — header, panel, arena, editor overlays, control bar
3. **Tile definitions** (~350 lines) — tile drawing functions, palette data
4. **Tile rendering** (`compileArena()`) — renders all tiles to offscreen canvas for sensor sampling
5. **Map editor** (~200 lines) — place/erase/rotate tiles, save/load JSON
6. **Camera** — zoom, pan, follow mode
7. **Robot config & state** — `cfg` object (sliders), `robot` object (runtime state)
8. **Physics** (~200 lines) — `computeTilt()`, `physicsStep()`, `checkSensorClearance()`
9. **Sensor model** (~100 lines) — `refl()`, `pxColor()`, `sensorHeightDiff()`, far-field blur
10. **Arena rendering** (~150 lines) — `drawArena()`, `drawRobot()`, PiP pitch/roll views
11. **Blockly/Code editor** (~250 lines) — block definitions, toolbox, Python generation
12. **Skulpt runtime shim** (~200 lines) — `_simShim` bridges blocks to simulator physics
13. **Simulation loop** — `requestAnimationFrame` at ~60fps
14. **Config persistence** — localStorage save/load of all slider values

## Robot Config Object
```javascript
cfg = {
  wheelDiam: 56,      // drive wheel diameter (mm)
  axleTrack: 112,     // distance between drive wheels (mm)
  driveFwd: 0,        // drive axle offset from hub centre (mm, + = forward)
  colorFwd: 44,       // colour sensor forward offset (mm)
  colorSpread: 15,    // colour sensor lateral half-spread (mm)
  colorHeight: 20,    // colour sensor height above ground (mm)
  ultraFwd: 44,       // ultrasonic sensor forward offset (mm)
  passiveFwd: -44,    // passive wheel fore/aft (mm, - = behind axle)
  passiveLat: 0,      // passive wheel lateral offset (mm)
  imuDrift: 0         // IMU heading drift (deg/min)
}
```

## Physics Model

### Heading convention
- 0° = north (up), 90° = east (right), clockwise positive
- `omega = (vL - vR) / axleTrack` — **left minus right** for clockwise-positive

### Body plane (3-point tilt)
Two drive wheels + one passive wheel, each elevated by wheel radius above ground contact. `computeTilt()` calculates pitch and roll slopes. The drive axle is the pivot — all offsets (sensor, passive wheel) are relative to it.

### Sensor height degradation
`sensorHeightDiff()` returns deviation from ideal working distance. The `refl()` function uses a far-field light-cone blur model: as deviation increases, readings blend toward background colour. At ~4.3mm deviation, the sensor goes blind. See `docs/colour-sensor-height-model.md` for full details.

### Sensor ground collision (stall)
`checkSensorClearance()` checks if either colour sensor's Z position is at or below ground level. If so, `physicsStep()` reverts the robot's position — the robot stalls and turns red.

### Ramp speed factors
```javascript
RAMP = {
  none:  { sf: 1.0 },
  up:    { sf: 0.7 },
  down:  { sf: 1.3 },
  crest: { sf: 0.85 }
}
```

## Conventions & Patterns

### Code style
- Minified-ish single-line style throughout (semicolons, short variable names)
- No modules, no classes — plain functions and global state
- DOM manipulation via `document.getElementById()` and inline `onclick` handlers

### Editing tips
- Search for `// ===` section headers to navigate the file
- `cfg.` prefix → slider-configurable robot parameter
- `robot.` prefix → runtime simulation state
- The `_simShim` object bridges Skulpt Python calls to simulator internals

### Testing
- No automated tests. Verify visually by running the simulator.
- Use the demo modes (Tuned P, Basic P, Bang-Bang) to smoke-test physics changes.
- For sensor/ramp changes, fast-forward the sim and check the robot follows the line through ramp transitions.

## File Map

```
robocup-simulator.html    # The entire application (HTML + CSS + JS)
docs/
  user-manual.md          # Full user manual (markdown source)
  user-manual.html        # HTML wrapper that renders the markdown
  images/                 # Screenshots for the manual
  colour-sensor-height-model.md  # Technical doc on sensor physics
tiles/                    # Pre-rendered tile PNG images (loaded at startup)
rcjl_map.json             # Default arena map
rcjl_map_with_elevation.json  # Multi-level arena map
.claude/launch.json       # Dev server config (python -m http.server 8765)
```

## What Works (v0.4)
- Differential drive physics with configurable wheel geometry
- Configurable drive axle offset (moves pivot point forward/back)
- 3-point body tilt on ramps with realistic pitch/roll
- Colour sensor height degradation (far-field blur model)
- Sensor ground collision → robot stall
- Multi-level arena (L0-L3) with ramp tiles
- Map editor with 80+ tile types, save/load JSON
- Blockly visual code editor generating Pybricks Python
- Skulpt Python runtime with DriveBase, Motor, ColorSensor, UltrasonicSensor, IMU shim
- Save/load block programs (XML files)
- Camera zoom/pan/follow mode
- PiP pitch and roll views
- User manual with screenshots

## Relevant External References
- RCJA 2026 rules: https://www.robocupjunior.org.au/wp-content/uploads/2026/02/RCJA-Rescue-Line-Rules-2026.pdf
- Tile downloads: https://rcja.app/rcj_cms/rescue/tiles
- Pybricks API: https://docs.pybricks.com/en/latest/
- SPIKE Prime hub: 88×56×32mm
- Colour sensor optimal distance: 16mm
- Ramp max incline: 25° (2026 rules)
- Level height: 90mm per level
