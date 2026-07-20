# Parametric-Radial-Tessellation-SVG-Generator
Access the generator here" https://programmadan.github.io/Parametric-Radial-Tessellation-SVG-Generator/

Tessela, the low hanging fruit of a name that is all too perfect a fit a for a Claude (Fable: high) implemented algorithm for the generation of tessellations. Heh, it even rhymes. 
[RANT WARNING] In all seriousness, this is a good example of a usecase which doesn't suffer from the lack of readability and organization that is often characteristic of generated code (though even this can controlled with framework instructions). Tessela is the bridge between an idea and its final product, in this case a downloadable vector graphic of any tessellation it generates, which can be used for graphic design. At least that was my motivation for this project, the ability to generate cool tessellated patternsd to spice up the blank space in my website. It's important to note that hade I simply asked a model to create a tessellation generator, without specifying an exact algorithm, the results would be rather unpredictable. I had a clear vision, and confidence that it would work, but with AI as the bridge between idea and reality, I need not expend the time or energy bringing it to life myself. Don't get me wrong, there is a time and place to test your own skills, but when we sometimes get an idea or require a tool, why choose anything but the fastest way to get there? Boldy put, it doesn't matter whether the code for an isolated tool is spaghetti, as long as the tool works. And Tessela does indeed work. Now I know what you're thinking. You can hardly call something this hard to edit "open source". But there is still a way. You have access to both the source code and spec.md file, meaning another model can pick up where mine left off. Admittedly it won't be the most token stingy process, but that's how you'd do it. 

Tessela is a single-file browser app that generates radial tessellations, patterns of polygons that fill a disc outward from a center point and allows you edit them by hand before exporting to SVG.
## Getting started

Open `index.html` in any modern browser, or click on the link above to use without downloading. Everything runs locally and nothing is saved between sessions, so **export before you close the tab**.

Click **Generate** to produce your first pattern, then adjust parameters and regenerate until you like the result.

## The workflow

1. **Generate** a pattern from parameters (Settings panel).
2. **Edit** individual cells if you want to fine-tune.
3. **Export SVG** for use in a vector editor, plotter, laser cutter, or wherever else.

Generation is random but reproducible: leave the seed blank for a new pattern each time, or type a number to get the exact same pattern back. The seed of every generated pattern is shown in the status bar, so you can note down one you like.

## Parameters

Open the **Settings** panel to reach these. All of them apply at generation time — changing one does not alter a pattern already on screen until you press Generate again.

### Structure

| Parameter | What it does |
|---|---|
| **Number of tiers** | How many concentric rings of cells make up the disc. More tiers = larger, more complex pattern. |
| **Initial radial width** | Thickness of the innermost ring, in document units. Sets the overall scale. |
| **Tier scale factor** | How much each ring's thickness grows over the one inside it. `1.0` = uniform rings; `1.35` = each ring is 35% thicker than the last, so the pattern opens up as it goes outward. |

### Cell size and shape

| Parameter | What it does |
|---|---|
| **Initial vertex density** | How finely the center is subdivided. Higher = smaller, more numerous cells. |
| **Initial / final ratio** | How much larger cells get toward the rim. `1` = same size throughout; `3` = rim cells are roughly three times the size of center cells. |
| **Irregularity (0–1)** | How much randomness is allowed in cell shape. `0` aims for clean, regular polygons; `1` produces a loose, organic, hand-drawn feel. |
| **Polygon distribution** | A table of how often each polygon type should appear — e.g. 45% triangles, 30% quadrilaterals, 25% pentagons. Add or remove rows with **+ Add polygon type**. The percentages must sum to 100 before you can generate. |

The distribution is a *target*, not a promise. Filling a disc with polygons is a constraint problem, and the generator repairs cells that won't fit — usually by splitting them into triangles. Expect the achieved mix to skew toward triangles compared to what you asked for. The readout below the settings shows target versus achieved side by side, so you can see how far off it landed and adjust.

### Radial elongation

A final shaping pass, applied to the whole pattern once it's generated.

| Value | Effect |
|---|---|
| **1** | Off. Cells keep their natural proportions. |
| **> 1** | Cells stretch along the radius and narrow across it — diamonds become slender **shards** radiating from the center. |
| **< 1** | The opposite: cells flatten into wide, **arc-like** bands wrapping around the center. |

The effect is proportional, so it looks consistent everywhere: a factor of 3 makes every cell three times as elongated relative to its width, whether it sits near the center or at the rim. The range is 0.1 to 10, and the outer edge of the pattern stays put regardless.

Two things worth knowing. Values well above 1 also make the center cells *smaller* overall, not just thinner — that's an unavoidable consequence of stretching cells radially when they share corners, and it compounds the size gradient you set with "initial / final ratio". And at the far extremes, the pattern is under enough strain that a couple of edges may end up crossing. The app repairs these automatically where it can and tells you in the settings readout when anything is left over; a different seed or a gentler factor usually clears it.

### Appearance

| Parameter | What it does |
|---|---|
| **Black fill probability** | Roughly what fraction of cells are filled black. No two black cells will ever touch, so very high values can't be fully reached — the app fills as many as the rule allows and reports the shortfall. |
| **Stroke width** | Outline thickness, in document units. Not scaled by anything else, so it's the one control for how heavy the linework looks. |
| **Seed** | Blank for random; any number to reproduce a specific pattern exactly. |

## Editing

Click a cell, edge, or vertex to select it. The status bar tells you what's selected and what you can do with it. Press **Escape** to deselect.

| Tool | How to use it |
|---|---|
| **Add vertex** | Select an edge, then click this to split it at its midpoint. |
| **Add edge** | Click one vertex, shift-click a second, then click this to connect them. The new edge must stay inside the pattern and can't cross anything. |
| **Delete** | Removes the selected item. Deleting an edge merges the two cells on either side; deleting a boundary edge removes its cell entirely. |
| **Paint** | Click to enter paint mode, then click cells to recolor them. A color picker offers black, white, and three accent colors. |

You can also **drag a vertex** to move it, and **right-click** anything for a context menu of the actions available to it.

Edits that would break the pattern — a crossing edge, a merge that would leave a hole — are rejected with a brief explanation rather than silently applied.

### Navigation and shortcuts

- **Scroll** to zoom, **drag the background** to pan.
- **Ctrl/Cmd + Z** to undo, **Ctrl/Cmd + Y** (or Ctrl/Cmd + Shift + Z) to redo. Up to 100 steps.
- **Delete** or **Backspace** deletes the selection.
- **Escape** clears the selection and exits paint mode.

Regenerating discards your edits, so the app asks for confirmation first if you've made any.

## Exporting

**Export SVG** downloads `tessellation.svg`. Inside it:

- Each cell is a closed `<path>`, grouped by tier (`tier-0`, `tier-1`, …) so you can select whole rings at once in a vector editor.
- Outer boundary edges that don't belong to a cell are written as separate `<line>` elements.
- The viewBox is cropped tight to the artwork, with a small margin so strokes aren't clipped.
- All fills and strokes are plain attributes — no CSS, no external references — so it opens cleanly in Illustrator, Inkscape, Figma, and cutting or plotting software.

## Limitations

- Nothing persists between sessions. Export or lose it.
- The polygon distribution is a bias, not a guarantee (see above).
- Black fill can't reach very high percentages without breaking the no-touching rule.
- Radial elongation at the extremes can leave a small number of crossing edges, which the app reports rather than hides.
- Very high tier counts with high density produce thousands of cells and will feel sluggish to edit.
