# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Development

No build system. Open `index.html` directly in a browser, or serve locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Deployment is via GitHub Pages — pushing `index.html` to the repo root is sufficient.

## Architecture

The entire app lives in a single `index.html` file with four self-contained sections:

**CSS (~lines 9–195):** All layout and component styles are inline. `z-index` stacking (low → high):
- `#dog-wrap` (100) → `#monkey-wrap` (205) → `#cam-wrap` (200) → `#cam-toggle` (201) → `#condiment-panel` (250) → `#dialog-box` (300) → `#cursor-dot` (9999)

**HTML (~lines 160–430):** In order:
- `#cursor-dot` — custom cursor dot
- `#particle-canvas` — full-screen canvas for particles and poop
- `#monkey-wrap` — SVG monkey character (clickable, right side of screen)
- `#dog-wrap` — SVG hotdog-dog (follows cursor)
- `#condiment-panel` — 5 bottle buttons (ketchup, mustard, wasabi, soy, pepper)
- `#cam-wrap` / `#cam-toggle` — webcam feed + emotion tag
- `#dialog-box` — pixel chat input/output

**SVG dog (~lines 167–231):** Horizontal hotdog-dog, `viewBox="0 0 280 170"`, face on LEFT end. Key IDs:
- `#m1`, `#m2` — mustard gradient stops updated by `applyTheme(name)` to reflect facial emotion
- `#dog-svg` — transformed entirely via `style.transform`; never touch SVG attributes at runtime

**SVG monkey (~lines 255–319):** Front-facing cartoon monkey, `viewBox="0 0 80 115"`.
- `#mk-siren-glow` — red circle whose `opacity` and `fill` are pulsed in JS during chase
- All animation via `monkeySvg.style.transform`; no SVG attribute mutations

---

## JavaScript Module (~lines 432–end)

Single `<script type="module">`. No bundler. One CDN import: MediaPipe `vision_bundle.mjs`.

### Emotion themes

```
THEMES  →  { default, happy, angry, disgusted, surprised, sad }
```
Each theme stores mustard gradient colors (`m1`, `m2`) and glow RGB values. `applyTheme(name)` writes to `#m1`/`#m2` stop-color attributes and updates `#emotion-tag`.

---

### Dog state machine

`mode` variable + `modeT` timer. `enterMode(m)` resets timer, clears shake, resets poop/sneeze flags, and sets glow. Allowed re-entry for `'stunned'` (monkey hit can re-trigger).

| Mode | Trigger | Behaviour |
|---|---|---|
| `idle` | default | follows cursor with breath scale |
| `run` | tap on dog | runs to random targets for 3.6 s |
| `belly` | stroke across dog | rotates 88°, drifts to cursor |
| `spin` | circular cursor motion | spins 540°/s for 1.5 s |
| `happy_react` | ketchup | bounces joyfully for 2 s |
| `dislike_react` | mustard | rapid lateral shake for 1.6 s |
| `poop_run` | wasabi | frantic run dropping green poop every 0.42 s for 4.2 s |
| `poop_squat` | soy sauce | squats and drops 2 brown poops, springs back in 1.8 s |
| `sneeze` | pepper | lateral head bob + particle bursts (5 max) for 1.6 s |
| `stunned` | monkey bonk | squish/wobble for 1.5 s; stars burst from head |

**`applyTransform({bounceY, rotation, scaleY, overrideX})`** — single bottleneck for `#dog-svg` transform. Combines `facingLeft` flip with `stretchX` scalar:
```js
const fx = facingLeft ? -1 : 1;
const sx = overrideX !== null ? overrideX : fx * stretchX;
dogSvg.style.transform = `rotate(${rotation}deg) translateY(${bounceY}px) scale(${sx},${scaleY})`;
```

**`stretchX`** — grows while mouse is held (rate 2.2×/s, max 3.8×), springs back on release. Disabled in belly/spin/react/stunned modes via `canStretch` guard.

**`shakeOffsetX`** — applied to `dogWrap.style.left` separately for lateral displacement (dislike shake, sneeze bob).

**Poop timing trick** (frame-rate-independent periodic drops):
```js
if (Math.floor(modeT/interval) > Math.floor((modeT-dt)/interval)) spawnPoop(…);
```

---

### Monkey state machine

`monkeyMode` variable + `mkT` timer. `enterMonkeyMode(m)` resets timer and flags. Position tracked as `(monkeyX, monkeyY)`; home position is `(innerWidth−130, innerHeight/2)`.

| Mode | Behaviour |
|---|---|
| `idle` | idles at home with slow bob, facing left; pointer-events on |
| `spinning` | full-body spin (820°/s) for 0.9 s before chasing |
| `chasing` | moves toward dog at 14% lerp/frame; siren pulses; run bounce |
| `hitting` | lunges forward (sin arc rotate), triggers `hitDog()` at t=0.22 s |
| `returning` | runs back home after hitting |

**`hitDog()`** — calls `enterMode('stunned')`, bursts yellow star particles, and spawns a DOM `BONK!` label that floats and fades.

**Siren glow** — `#mk-siren-glow` opacity and fill are modulated by `Math.sin(sirenT*9)` during chase states; off otherwise.

---

### Particle + Poop system (canvas)

Both run in `renderLoop()` on `#particle-canvas` (z-index 150, pointer-events none).

**`Particle(x, y, vx, vy, color, r)`** — physics dot with gravity (vy += 0.28/frame), air drag (vx *= 0.97), life decay. `burstAt(x, y, color, count, speed)` spawns a radial burst.

**`Poop(x, y, color)`** — multi-tier pile (3 ellipse tiers + swirl tip + tiny eyes + shine). `decay = 0.0028` (~6 s lifetime). Green (`#3d7a12`) for wasabi; brown (`#3d1e08` / `#2e1606`) for soy sauce.

**`spawnPoop(x, y, color)`** — pushes a new `Poop` to the `poops` array.

---

### Condiment system

Five bottles in `#condiment-panel`. Drag over dog to trigger reaction (1800 ms cooldown). Each bottle has `data-cond` attribute; a single handler loop wires all five.

| Condiment | Color | Dog reaction |
|---|---|---|
| ketchup | red | `happy_react` + red burst |
| mustard | yellow | `dislike_react` + yellow burst |
| wasabi | green | `poop_run` (green poop) |
| soy sauce | brown | `poop_squat` (brown poop) |
| pepper | grey | `sneeze` (white mist + dark specks) |

`emitSpray(condiment, btnEl)` fires 4 particles per frame from bottle tip toward cursor while dragging. `triggerReaction(cond)` fires once when spray overlaps dog hit box (`DOG_HIT_W=110, DOG_HIT_H=55`).

---

### MediaPipe face detection

`initFace()` loads `FaceLandmarker` (float16, GPU delegate) from CDN. Runs `detectForVideo` every 6th animation frame. `classify(categories)` computes weighted scores for 5 emotions and calls `applyTheme` when the dominant emotion changes (threshold 0.30). Suppressed during active reaction/stun modes. Gracefully hides cam UI on permission denial or load failure.

---

### Pixel chat box

`DOG_REPLIES` keyed by animal name substring. User message → triggers `run` mode → dog types reply character-by-character (42 ms/char) with a blinking cursor span. `chatInput` mousedown/touchstart stop-propagated to prevent interfering with dog interactions.
