# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build system. Open `index.html` directly in a browser, or serve it locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Deployment is via GitHub Pages — pushing `index.html` to the repo root is sufficient.

## Architecture

The entire app lives in a single `index.html` file with three self-contained sections:

**CSS (lines ~9–192):** All layout and component styles are inline. Key visual layers use fixed `z-index` stacking: cursor dot (9999) → dialog box (300) → cam toggle (201) → cam wrap (200) → dog wrap (100).

**SVG dog (lines ~200–319):** An inline SVG hotdog-dog character. The mustard squiggle (`#mustard-path`) uses two gradient stops (`#m1`, `#m2`) that are mutated at runtime to reflect detected facial emotion. The `#dog-svg` element is scaled via JS `style.transform` — never via SVG attributes.

**JavaScript module (lines ~343–795):** A single `<script type="module">` with no bundler or imports except MediaPipe from CDN. Key systems:

- **Emotion themes (`THEMES`):** Maps 5 emotions + default to mustard gradient colors and glow RGB values. `applyTheme(name)` updates the SVG gradient stops and the `#emotion-tag` label.

- **Dog state machine:** Four modes — `idle`, `run`, `belly`, `spin`. `enterMode(m)` transitions between them; each has a `tick*` function called every animation frame. `stretchX` is a separate scalar applied on top of the mode transform — it grows while the mouse/touch is held and springs back on release.

- **`applyTransform(opts)`:** Single function that assembles and writes the final `style.transform` on `#dog-svg`. All rotation, bounce, stretch, facing direction, and scale changes go through here.

- **Circular motion detector (`detectCircle`):** Accumulates angle deltas of the cursor around the dog; triggers `spin` mode when the total exceeds ~1.7π radians.

- **MediaPipe face detection (`initFace` / `classify`):** Loads `FaceLandmarker` from CDN, runs every 6th animation frame on the webcam feed, and calls `applyTheme` when the dominant emotion changes. Hides the cam UI gracefully if the model fails to load or camera permission is denied.

- **Pixel chat box:** Hardcoded `DOG_REPLIES` keyed by animal name substring. Typing a message triggers `run` mode; the dog "types" its reply character-by-character with a blinking cursor span.
