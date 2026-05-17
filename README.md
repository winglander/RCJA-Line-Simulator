# RCJA Line Rescue Simulator

Browser-based simulator for RoboCup Junior Australia Rescue Line. Students write Pybricks-style Python to control a simulated SPIKE Prime robot that follows lines on a tile-based arena.

[RCJA-Line-Rescue-Simulator](https://winglander.github.io/RCJA-Line-Simulator/)
- Single HTML file (`robocup-simulator.html`), no server required
- Runs on desktop and mobile browsers
- Canvas 2D rendering, vanilla JS

## Colour Sensor Noise Model

### Problem

On ramps and ramp transitions, the colour sensor's distance from the tile surface changes, degrading readings. The noise should be **zero** when the robot is fully on any uniform surface (flat or ramp) and **highest** during transitions where wheels straddle different surfaces.

### Rigid-Plane Body Model

The robot body is modelled as a rigid plane defined by three contact points:

```
  Robot local coordinate frame (top-down view, heading = up)

         ^ +fwd
         |
    LW --+-- RW        drive wheels at (±axleTrack/2, 0)
         |
         * hub centre (0, 0)
         |
    SL --+-- SR        colour sensors at (±colorSpread, colorFwd)
         |
         o PW           passive wheel at (passiveLat, passiveFwd)
```

All positions are configurable via sliders. The passive wheel can be in front of or behind the drive axle.

Each contact point has a world Z elevation sampled from the terrain:

```
zL = elevAt(leftWheel)      left drive wheel
zR = elevAt(rightWheel)     right drive wheel
zP = elevAt(passiveWheel)   passive/caster wheel
```

### Plane Equation

The three contact points define a tilted plane. For any point at robot-local position `(x, y)`, the Z elevation on the body plane is:

```
T = axleTrack

a = (zR - zL) / T                            roll gradient (lateral tilt)
c = (zL + zR) / 2                            drive axle midpoint elevation
b = (zP - a * passiveLat - c) / passiveFwd   pitch gradient (fore-aft tilt)

bodyZ(x, y) = a * x + b * y + c
```

This captures both **pitch** (fore-aft tilt from ramp slope) and **roll** (left-right tilt from uneven wheel elevations).

### Sensor Height Calculation

Each colour sensor's height above the ground surface beneath it:

```
sensorBodyZ = a * (±colorSpread) + b * colorFwd + c
groundZ     = elevAt(sensorWorldX, sensorWorldY)
heightDiff  = |sensorBodyZ - groundZ|
```

Where `sensorWorldX/Y` is the sensor's world position computed from robot heading.

### Noise from Height Deviation

```
noise = min(0.5, heightDiff / 50)
reflection = clamp(baseLuminance + (random() - 0.5) * noise * 100, 0, 100)
```

The divisor (50mm) and cap (0.5) control sensitivity. At 25mm deviation the noise factor is 0.5 (max), adding up to ±25 to the 0-100 reflection reading.

### Why This Works

| Scenario | heightDiff | Noise | Reason |
|----------|-----------|-------|--------|
| Flat ground | 0 | 0 | All points on same plane, sensor parallel to surface |
| Fully on ramp | ~0 | ~0 | All points on ramp plane, sensor still parallel to surface |
| Ramp entry | high | high | Drive wheels on ramp, passive on flat (or vice versa) - body tilts relative to surface under sensor |
| Ramp exit | high | high | Same mismatch in reverse direction |

The key insight is that **tilt alone does not cause noise**. A robot fully on a 15-degree ramp has 15 degrees of tilt but near-zero noise because the sensor-to-surface distance is unchanged. Noise only occurs when the body plane and the ground surface beneath the sensor diverge - which happens during transitions between surfaces at different elevations.

### Roll Effect

Because the model uses the full 3-point plane (not just pitch), a sensor on the **higher side** of a lateral tilt will be further from the ground than a sensor on the lower side. This means left and right colour sensors can have **different noise levels** in the same tick - matching real-world behaviour when one wheel rides up a ramp edge before the other.

## Physics Design

See [physics-design.md](physics-design.md) for the full simulation design document covering kinematics, elevation, sensors, and the simulation loop.
