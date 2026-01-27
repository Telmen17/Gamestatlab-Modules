# Projectile Motion Code Explanation

## Overview
The `animateBall()` function simulates a soccer ball being kicked using physics equations. Here's how it works:

## Step-by-Step Breakdown

### 1. **Function Parameters** (Line 584)
```javascript
function animateBall(startPercentX, startY, endPercentX, endY, duration, callback)
```
- `startPercentX`: Starting X position as percentage (e.g., 52 = 52% from left)
- `startY`: Starting Y position in pixels from BOTTOM (CSS bottom coordinate)
- `endPercentX`: Ending X position as percentage
- `endY`: Ending Y position in pixels from BOTTOM
- `duration`: Animation time in milliseconds (900ms = 0.9 seconds)
- `callback`: Function to call when animation completes

### 2. **Coordinate System Conversion** (Lines 590-598)

**THE KEY ISSUE**: CSS uses a "bottom-up" coordinate system, but physics uses "top-down".

```javascript
// Convert percentages to pixels
const startX = (startPercentX / 100) * areaWidth;  // e.g., 52% → 312px (if width=600px)
const endX = (endPercentX / 100) * areaWidth;

// Convert CSS bottom coordinates to physics coordinates
const startY_physics = areaHeight - startY;
const endY_physics = areaHeight - endY;
```

**Example:**
- If `areaHeight = 400px` and `startY = 365px` (35px from bottom)
- Then `startY_physics = 400 - 365 = 35px` (35px from top in physics)
- This means the ball starts LOW in physics coordinates

### 3. **Physics Constants** (Lines 600-602)
```javascript
const g = 500;  // Gravity: 500 pixels per second²
const T = duration / 1000;  // Total time in seconds (0.9s)
```

### 4. **The Problem: Peak Height Calculation** (Lines 608-618)

**CURRENT CODE:**
```javascript
const distanceY_physics = endY_physics - startY_physics;
const peakHeight = Math.max(80, Math.abs(distanceY_physics) + 60);
const vy0 = Math.sqrt(2 * g * peakHeight);
```

**THE BUG**: This calculates peak height from the STARTING position, not from ground level!

**What's happening:**
- If `startY_physics = 35px` (ball is 35px from top = near bottom)
- And `peakHeight = 80px`
- Then `vy0 = sqrt(2 * 500 * 80) = sqrt(80000) = 282 px/s`

**The problem**: The peak height is calculated as 80px ABOVE the starting position, but the starting position is already 35px from the top. So the ball might not actually go up much relative to where it starts.

### 5. **Horizontal Velocity** (Line 621)
```javascript
const vx0 = distanceX / T;
```
This is simple: distance divided by time. If ball needs to travel 200px in 0.9s, vx0 = 222 px/s.

### 6. **The Animation Loop** (Lines 626-662)

**Time tracking:**
```javascript
const elapsed = (currentTime - startTime) / 1000;  // Convert to seconds
```

**Position calculation:**
```javascript
// Horizontal: simple linear motion
const currentX = startX + vx0 * elapsed;

// Vertical: projectile motion equation
// y(t) = y₀ + v₀t - ½gt²
const currentY_physics = startY_physics + vy0 * elapsed - 0.5 * g * elapsed * elapsed;
```

**Side curve (optional):**
```javascript
const progress = elapsed / T;  // 0 to 1
const sideCurve = Math.sin(progress * Math.PI) * curveAmount * (1 - progress);
const curvedX = currentX + sideCurve;
```

**Convert back to CSS:**
```javascript
const currentY = areaHeight - currentY_physics;  // Convert physics Y back to CSS bottom
const clampedY = Math.max(currentY, 30);  // Don't let ball go below ground
```

## THE MAIN PROBLEMS

### Problem 1: Peak Height Calculation
The peak height is calculated from the starting position, not from ground level. If the ball starts at `startY_physics = 35px`, and we want it to peak at 80px above that, the actual peak would be at 35 + 80 = 115px from top.

But if the goal is at `endY_physics = 200px` (200px from top), the ball needs to go UP from 35px to reach 200px, which requires a different calculation.

### Problem 2: Velocity Doesn't Match End Position
The code calculates `vy0` to reach a peak, but doesn't ensure the ball actually reaches the end position. The ball might peak too early or too late.

### Problem 3: Coordinate System Confusion
- CSS `bottom: 35px` means 35px from bottom (ball is LOW)
- Physics `y = 35px` means 35px from top (ball is HIGH)
- The conversion `startY_physics = areaHeight - startY` handles this, but it's easy to get confused.

## HOW TO FIX IT

### Option 1: Calculate Peak Based on Trajectory
Instead of arbitrary peak height, calculate what peak is needed to reach the end position:

```javascript
// Calculate time to reach end position
// We need: endY_physics = startY_physics + vy0*T - 0.5*g*T²
// Rearrange: vy0 = (endY_physics - startY_physics + 0.5*g*T²) / T

// But we want the ball to go UP first, so ensure vy0 is positive and large enough
const vy0_needed = (endY_physics - startY_physics + 0.5 * g * T * T) / T;
const minVy0 = 200;  // Minimum upward velocity
const vy0 = Math.max(vy0_needed, minVy0);

// Now calculate actual peak
const t_peak = vy0 / g;  // Time to reach peak
const peakY_physics = startY_physics + vy0 * t_peak - 0.5 * g * t_peak * t_peak;
```

### Option 2: Ensure Ball Always Goes Up First
Make sure the initial velocity is always upward and significant:

```javascript
// Calculate what vy0 is needed to reach end position
let vy0 = (endY_physics - startY_physics + 0.5 * g * T * T) / T;

// If end is LOWER than start, we still want upward arc
// So ensure minimum upward velocity
if (vy0 < 150) {
    vy0 = 150;  // Force minimum upward velocity
}

// Recalculate end position if we changed vy0
const actualEndY_physics = startY_physics + vy0 * T - 0.5 * g * T * T;
```

### Option 3: Start from True Ground Level
Make sure the ball always starts from the same ground level:

```javascript
// Always start from ground (e.g., 30px from bottom)
const GROUND_LEVEL = 30;
const startY = areaHeight - GROUND_LEVEL;  // Always same starting height
const startY_physics = areaHeight - startY;  // = GROUND_LEVEL
```

## Debugging Tips

1. **Add console.logs:**
```javascript
console.log('Start:', {startX, startY_physics, startY});
console.log('End:', {endX, endY_physics, endY});
console.log('Velocities:', {vx0, vy0});
console.log('At t=0.1s:', {
    elapsed: 0.1,
    x: startX + vx0 * 0.1,
    y_physics: startY_physics + vy0 * 0.1 - 0.5 * g * 0.1 * 0.1,
    y_css: areaHeight - (startY_physics + vy0 * 0.1 - 0.5 * g * 0.1 * 0.1)
});
```

2. **Check the starting position:**
Make sure `ball.style.bottom` is set correctly before animation starts.

3. **Visualize the trajectory:**
Calculate positions at different times and see if they make sense.

## Summary

The main issue is that the peak height calculation doesn't account for the actual trajectory needed to reach the end position. The ball might be calculating a peak that doesn't align with where it needs to end up, causing it to appear to "drop from above" instead of arcing upward first.
