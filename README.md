# Parabola Graphing Interactive

**Live Site Link:** https://content-interactives.github.io/parabola_graphing/

A React single-page interactive on a Cartesian grid. Students place a **vertex** and a **second point** (both snapped to integer coordinates); the app draws the unique parabola \(y = a(x-h)^2 + k\) through those constraints, clipped to the viewport with **direction arrows** at the visible endpoints. Multiple parabolas are supported with the same **two-point / one-parabola** rhythm as other interactives in this series; at most **two** completed curves stay visible, with undo/redo over the full point history.

## Tech stack

| Layer | Choice |
|--------|--------|
| UI | React 19 |
| Build | Vite 7 with `@vitejs/plugin-react` |
| Styling | Root layout inline; `src/glow.css` for segmented Undo / Redo / Reset |
| Deploy | `gh-pages` publishing `dist/` |

Production builds assume hosting under **`/parabola_graphing/`** (`vite.config.js` → `base`).

## Repository layout

```
src/
  main.jsx
  App.jsx                 # Renders ParabolaGraphing
  index.css, App.css
  glow.css                  # Segmented “glow” buttons
  components/
    ParabolaGraphing.jsx    # Grid, parabola geometry, animation, history
```

## Coordinate system

- **Logical domain**: \([-10, 10]\) on both axes for **placed** points; ticks match integers in that range.
- **Canvas**: `500×500` SVG pixels; `PADDING = 40` for the inner plot.
- **Mapping**: `valueToX` / `valueToY` (y flipped for SVG); `xToValue` / `yToValue` for pointer ↔ model.
- **Snapping**: Clicks and hover preview use `roundToTick` after clamping to \([-10,10]\).

Axis geometry uses `EXTENDED_MIN` / `EXTENDED_MAX` (one unit beyond the labeled range) so arrowheads sit past the outer ticks.

**Extended value bounds** (`clampExtended`) and **container-aligned value bounds** (`containerMinX` … `containerMaxY`) map the SVG rectangle to model space so parabolic arcs can be clipped to the **top and bottom** of the graph like infinite lines in the line-drawing interactives.

## Parabola model

For vertex \((h,k)\) and second point \((x_2, y_2)\):

\[
a = \frac{y_2 - k}{(x_2 - h)^2}
\]

If \(|x_2 - h| < \) **`MIN_DX`** (0.5), the parabola is not constructed (avoids division by zero and near-degenerate width).

## `parabolaPathAndArrows(h, k, a)`

Pure helper (value-space coefficients → SVG). Responsibilities:

1. **Left / right branches** (`pathDLeft`, `pathDRight`): March from the vertex toward `containerMinX` and `containerMaxX`. If \(y(x)\) exits the vertical container band \([y_{\min}, y_{\max}]\) in value space, the path **stops at the analytic intersection** with \(y = y_{\text{bound}}\) (using \((y_{\text{bound}} - k)/a\) and \(\sqrt{\cdot}\) with correct branch left/right of \(h\)), so the curve does not flatten along the clipping edge incorrectly.
2. **Full path** (`pathD`): Used for the **ghost** preview; scans \(x\) with `PARABOLA_SAMPLES` segments, emitting `M`/`L` chains only while \(y\) is inside the vertical band, optionally closing partial spans at boundaries (supports re-entry if the parabola dipped out and back in within the horizontal window).
3. **Arrows**: Placed at the **actual** left/right stop points. Tangent in value space is \((1, 2a(x-h))\); `arrowDirFromTangent` maps that through `scaleX` / `scaleY` (with y negated) to a unit direction in SVG for `arrowPoints`.

Rendering uses two `<path>` elements (left and right of the vertex) so **stroke-dash animation** can run per branch.

## Interaction flow

1. **Click** trims any redo tail, appends a snapped point, and increments `historyIndex`. On each render, **current** points are `pointsHistory.slice(0, historyIndex)` (same undo/redo pattern as `CircleDrawing`).
2. **Odd** count → only the **vertex** is fixed for the in-progress parabola; **ghost** shows the curve through vertex + **hover** lattice point (dashed, semi-transparent), using `parabolaPathAndArrows` on the ghost `pathD`.
3. **Even** count ≥ 2 → a parabola **completes**; `animatingParabolaIndex` is set to `nextIndex/2 - 1` and `requestAnimationFrame` drives `parabolaProgress` over **`PARABOLA_ANIMATION_DURATION_MS`** (1500 ms).
4. **While animating**, clicks are ignored.

**Animation rendering**: For the active parabola, each branch path sets `pathLength={1}`, `strokeDasharray={1}`, `strokeDashoffset={1 - parabolaProgress}` so the stroke **grows** from the vertex outward. Arrow polygons appear only when **`parabolaProgress >= 1`**.

## Visibility window

- **`MAX_PARABOLAS = 2`**. `visibleParabolasFull` holds up to two non-animating parabolas; while one is animating, only **`MAX_PARABOLAS - 1`** full curves show alongside the animating one (mirrors `CircleDrawing` / `TwoPointDrawing` behavior).
- `visiblePointStartIndex` / `visibleParabolaStart` hide markers for parabolas that scrolled off the visual window when more than two exist in history.

## History: undo / redo / reset

Standard **array + index** pattern:

- `pointsHistory` holds all placed points; `historyIndex` is how many are active.
- **Undo** decrements `historyIndex`; if that cancels the second point of the animating parabola, animation state clears.
- **Redo** increments up to `pointsHistory.length` without discarding the tail.
- **Reset** clears history, index, progress, and animation.

Buttons use `stopPropagation()` so they do not place points.

## Accessibility

Root: `role="application"`, `aria-label="Parabola graphing coordinate plane"`, `tabIndex={0}`. SVG is non-interceptive (`pointerEvents: 'none'`); the wrapper `div` receives clicks.

## Scripts

| Command | Purpose |
|---------|---------|
| `npm run dev` | Vite dev server |
| `npm run build` | Output to `dist/` |
| `npm run preview` | Serve `dist/` locally |
| `npm run deploy` | Build then `gh-pages -d dist` |

GitHub Pages project URL pattern: `https://<user>.github.io/parabola_graphing/` to match `base`.

## Tunables

Constants at the top of `src/components/ParabolaGraphing.jsx`:

- `MIN` / `MAX`, `WIDTH` / `HEIGHT` / `PADDING`
- `MAX_PARABOLAS`, `MIN_DX`, `PARABOLA_SAMPLES`, `PARABOLA_ANIMATION_DURATION_MS`, `PARABOLA_ARROW_SIZE`, `POINT_RADIUS`

Clip path `plot-clip-parabola` keeps strokes inside the plot rectangle.
