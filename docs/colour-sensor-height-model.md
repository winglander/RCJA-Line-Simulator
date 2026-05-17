# Colour Sensor Height Degradation Model

## Problem

The simulator's colour sensor returned near-perfect readings regardless of the sensor's height above the ground surface. This meant the robot could traverse ramp-to-flat transitions without losing the black line — unrealistic behaviour, since real EV3/Spike colour sensors lose signal when lifted away from the surface.

### Root Cause

The original noise model applied a gentle linear noise factor capped at 50%:

```javascript
// OLD MODEL
function sensorNoise(lat) {
  // ...compute heightDiff...
  return Math.min(0.5, heightDiff / 50);
}
function refl(x, y, lat) {
  const n = sensorNoise(lat || 0);
  return clamp(trueLuminance + random() * n * 100);
}
```

At a typical ramp transition, `heightDiff` was ~12mm, giving `noise = 0.24` (a +/-24 swing). Black (0 +/- 24 = 0-24) and white (100 +/- 24 = 76-100) remained easily distinguishable — the robot never lost the line.

Additionally, the sensor had no mounting height — it was modelled as sitting at the body contact plane (0mm above ground). The physics also didn't account for wheel radius, placing the axle directly on the ground.

## Physics Model

### Wheel radius

The body plane is computed from three support points, each elevated by wheel radius above ground contact:

```javascript
function computeTilt() {
  const wr = cfg.wheelDiam / 2;  // drive wheel radius (default 28mm)
  const pr = wr;                  // passive mount matches drive wheel height
  const gL = elevAt(lx, ly), gR = elevAt(rx, ry), gP = elevAt(ppx, ppy);
  robot.groundElev = (gL + gR) / 2; // raw ground elevation (for level transitions)
  const zL = gL + wr, zR = gR + wr, zP = gP + pr; // axle heights
  const zD = (zL + zR) / 2;
  robot.elevation = zD;           // axle height (for PiP views)
  bodyPlane = {a: pa, b: pb, c: pc}; // c = zD (axle height at centre)
}
```

- `robot.groundElev` — raw ground contact elevation, used for level transitions and ramp detection
- `robot.elevation` — axle height (ground + wheelRadius), used for PiP terrain views
- `bodyPlane.c` — axle height at robot centre, equals `robot.elevation`

The passive wheel is physically small (8mm radius caster) but its mount height is set equal to the drive wheel radius so the chassis sits level on flat ground. The PiP pitch view draws the caster at its real 8mm size with a bracket connecting to the body plane.

### Sensor position

`cfg.colorHeight` is measured from the ground (not from the axle). On flat ground the sensor face sits at `colorHeight` mm above the surface. Since the body plane is at axle height (`ground + wheelRadius`), the sensor's offset relative to the body plane is:

```
sensorAboveBody = colorHeight - wheelRadius
```

For default values (colorHeight=20mm, wheelDiam=56mm): `sensorAboveBody = 20 - 28 = -8mm` — the sensor is mounted 8mm below the axle, pointing downward. This is physically correct for a downward-facing colour sensor.

## Geometry

When the robot is on a ramp with the colour sensor projecting over the flat upper-level tile:

```
                                    * S (sensor)
                                   /|
              colorHeight(20mm)   / |
                                 /  |  sensor-to-ground distance
                   axle plane   /---+  (bodyZ + sensorAboveBody - groundZ)
                               /    |
                              /     |  deviation from ideal
        =============== upper/level +  90mm ==============
                            /
                     * W   /  (wheel contact on ramp)
                          /       colorFwd offset
                         /        ------------------->
              ramp      /   8.6 deg
                       /
        ========= lower level 0mm ========================
```

### Height deviation calculation

```javascript
function sensorHeightDiff(lat) {
  const bodyZ  = bodyPlane.a * lat + bodyPlane.b * fwd + bodyPlane.c;
  const sensorAboveBody = cfg.colorHeight - cfg.wheelDiam / 2;
  const sensorZ = bodyZ + sensorAboveBody;
  const groundZ = elevAt(sx, sy);
  const dist = sensorZ - groundZ;        // actual sensor-to-ground distance
  return Math.abs(dist - cfg.colorHeight); // deviation from ideal
}
```

On flat ground:
- `bodyZ = 0 + 28 = 28mm` (ground + wheelRadius)
- `sensorZ = 28 + (-8) = 20mm`
- `groundZ = 0mm`
- `dist = 20mm`
- deviation = `|20 - 20| = 0` (no degradation)

On ramp transition (sensor over upper level):
- `bodyZ ≈ 102mm` (tilted body at sensor forward position)
- `sensorZ = 102 - 8 = 94mm`
- `groundZ = 90mm` (upper level)
- `dist = 4mm`
- deviation = `|4 - 20| = 16mm` (heavy degradation)

## Solution: Far-Field Light-Cone Blur Model

Real colour sensors emit a focused light cone. As the sensor lifts away from the surface, the cone diameter grows. When the illuminated spot exceeds the line width (~20mm), the sensor can no longer resolve the line — it reads the local area average (mostly background) instead of the surface directly below.

The simulator's offscreen canvas resolution is too low (~0.24 px/mm) for pixel-based spatial blur, so we use analytical far-field blending instead.

```javascript
function refl(x, y, lat) {
  const h = sensorHeightDiff(lat || 0); // deviation in mm
  const center = pxLum(x, y);           // luminance at sensor position

  // Near-ideal: tiny jitter only
  if (h <= 1) return clamp(center + (random() - 0.5) * 3);

  // Sample far-field background at ~50mm radius (4 points)
  const farPx = 50 * SCALE();
  let farSum = 0;
  for (let i = 0; i < 4; i++) {
    const a = (i / 4) * Math.PI * 2 + 0.3;
    farSum += pxLum(round(x + farPx * cos(a)), round(y + farPx * sin(a)));
  }
  const farAvg = farSum / 4;

  // Blend toward far-field as light cone grows past line width
  const spotMM = 2 * (1 + h * 1.5);                // cone diameter in mm
  const fade = min(1, max(0, (spotMM - 3) / 10));   // 0→1 over 3-13mm diameter
  const blended = center * (1 - fade) + farAvg * fade;
  const jitter = min(0.12, h * 0.02);
  return clamp(blended + (random() - 0.5) * jitter * 100);
}
```

### How it works

1. **Centre sample**: read luminance at the pixel directly under the sensor
2. **Far-field samples**: read 4 points at 50mm radius around the sensor (well beyond any line feature — these sample the background tile colour)
3. **Spot diameter**: `2 × (1 + h × 1.5)` mm — grows linearly with height deviation
4. **Fade factor**: `(spotDiameter - 3) / 10` — once the spot exceeds 3mm, the reading blends toward the far-field average. At 13mm diameter, fade = 1.0 (fully reading background)
5. **Result**: both left and right sensors read the same background average, so PID error → 0 and the robot drives straight off the line

### Degradation curve

| Deviation (mm) | Spot (mm) | fade | Black reads as | White reads as | Distinguishable? |
|---|---|---|---|---|---|
| 0 | 2.0 | 0 | ~0 | ~100 | Yes |
| 1 | 5.0 | 0.2 | ~18 | ~82 | Yes |
| 3 | 11.0 | 0.8 | ~40 | ~60 | Marginal |
| 4.3 | 14.9 | 1.0 | ~bg | ~bg | **Blind** |
| 16 | 50.0 | 1.0 | ~bg | ~bg | **Blind** |

At h=4.3mm deviation, the sensor is fully blind — both black and white read as background colour (~85-95 depending on tile).

### Colour detection (`pxColor`)

`pxColor()` also degrades — when deviation exceeds 6mm, colour detection returns `'NONE'`:

```javascript
function pxColor(x, y, lat) {
  const h = sensorHeightDiff(lat || 0);
  if (h > 6) return 'NONE';
  // ...normal colour classification...
}
```

## Configuration

The sensor height slider is in the **Colour Sensors** panel:

| Parameter | Range | Default | Unit | Meaning |
|---|---|---|---|---|
| Height above Ground | 5 - 40 | 20 | mm | Distance from ground to sensor face |

This is measured from the ground surface, not from the axle. Internally, the sensor's offset relative to the body plane (axle) is `colorHeight - wheelDiam/2`.

Lowering the height makes the sensor more resilient to elevation changes. Raising it makes ramp transitions harder — matching the trade-off in real robot design.

## Files Changed

- `robocup-simulator.html`
  - `computeTilt()` — wheel radius added to body plane computation; `robot.groundElev` for level transitions, `robot.elevation` for axle height
  - `sensorHeightDiff()` — deviation measured from ideal working distance (`cfg.colorHeight`), accounting for wheel radius
  - `refl()` — far-field light-cone blur model (replaces old noise model)
  - `pxColor()` — height-aware colour detection (returns `'NONE'` above 6mm deviation)
  - `cfg.colorHeight` — configurable sensor height (measured from ground)
