# Tessellation Editor — Implementation Specification (v2)

## Revision Notes (v1 → v2)

1. **Data model is now a true half-edge DCEL.** v1 called the structure a DCEL but stored undirected edges and unordered face-edge lists, which cannot support ordered boundary traversal, winding computation, or face splitting without ambiguity. v2 uses half-edges with `twin`/`next`/`prev` pointers.
2. **Generation now specifies collision handling.** v1's advancing-front algorithm had no overlap prevention and would self-intersect within a tier or two. v2 adds candidate validity tests, a spatial index, frontier snapping, retry/fallback rules, and gap-closure for merging fronts.
3. **The polygon type distribution is a bias, not a guarantee.** Closure regions are filled with whatever side counts the local geometry permits.
4. **The final tier has an explicit closure pass** (triangulate remaining gaps, then merge triangles upward toward the distribution) instead of an unenforced "no open edges may remain" assertion.
5. **Delete-edge is defined for faces sharing multiple edges** (contiguous chains merge; non-contiguous shared edges block the operation, since the merge would produce a face with a hole).
6. **Delete-edge on a boundary edge now deletes the incident face** rather than leaving an open, unrenderable face. Zero-face "naked" edges exist only as a transient artifact of Add-Edge between distinct boundary regions.
7. **Add-edge chords must lie strictly inside the face interior** — rejecting self-intersection alone is insufficient for concave faces.
8. **Winding is defined by the signed shoelace sum in SVG (y-down) coordinates**, removing the clockwise/positive-area ambiguity.
9. **Black fill greedy pass may fall short of target**; the shortfall is accepted and reported.
10. **"Rotating calipers" renamed to brute-force rotation search**, which is what the algorithm actually was.
11. **Boundary edges are no longer double-exported**: face strokes already cover one-face boundary edges; only zero-face naked edges are exported as `<line>` elements.
12. **Seed polygon construction specified** (v1 said generation starts "from a single seed point" without saying how the first face is made). The falloff constant `k` derivation is now given explicitly.
13. **(v2.2) Radial elongation parameter added, then redefined.** v2.1 implemented the literal reading — every radius multiplied by the factor — which is a similarity transform: it changes the exported size and stroke-to-cell ratio but leaves the pattern itself untouched. After clarification (cells themselves should thin into "shards"), v2.2 replaces it with the constant-anisotropy power map r′ = R_N·(r/R_N)^E, the unique radial-displacement map for which every cell's radial-to-tangential aspect changes by the same factor E at every radius. See the parameter's section for the derivation, guards, and repair pipeline.

---

## Overview

Build a browser-based tool for the algorithmic generation, interactive viewing, and manual editing of radial geometric tessellations. The output format is SVG. The application runs entirely in the browser with no backend.

---

## Data Model

The internal representation is a **half-edge DCEL (Doubly Connected Edge List)**. Vertices, half-edges, and faces are first-class entities. Faces do not store coordinate lists — geometry is reached only through shared vertices.

```
Vertex   { id: string, x: number, y: number, halfEdge: HalfEdgeID }
HalfEdge { id: string, origin: VertexID, twin: HalfEdgeID,
           next: HalfEdgeID, prev: HalfEdgeID, face: FaceID | null }
Face     { id: string, halfEdge: HalfEdgeID, fill: string, tier: number }
Graph    { vertices: Map<VertexID, Vertex>,
           halfEdges: Map<HalfEdgeID, HalfEdge>,
           faces: Map<FaceID, Face> }
```

Conventions and invariants:

- Every undirected edge is represented by exactly two half-edges that are each other's `twin`.
- `halfEdge.face` is the face lying to the **left** of the half-edge when traveling from `origin` to `twin.origin`. Interior face loops therefore traverse counterclockwise in mathematical (y-up) terms; see the Winding section for how this maps to SVG's y-down space.
- `face: null` means the half-edge borders unbounded space (or a hole). A **boundary edge** is an edge with exactly one non-null face across its two half-edges. A **naked edge** has `face: null` on both half-edges; naked edges arise only from explicit editor operations (see Add Edge / Delete Face) and are transient — the user is expected to resolve them.
- `next` and `prev` link half-edges into loops: face loops (all half-edges sharing a face) and boundary loops (`face: null` half-edges around holes and the outer perimeter). Every half-edge belongs to exactly one loop.
- `vertex.halfEdge` is any one half-edge with that vertex as origin. All half-edges around a vertex are reachable via `twin.next` orbits.
- Faces are simple polygons (no holes), possibly concave, with ≥ 3 edges.
- Every vertex has degree ≥ 1 (belongs to at least one edge). Isolated vertices are removed immediately when they arise.
- Shared vertices are unified — there is exactly one vertex object at any shared point. Moving it updates all incident half-edges and faces simultaneously.

Structural invariants to assert (in dev builds, after every mutation; in production, after generation and on export):

- `twin(twin(h)) === h` and `twin(h) !== h` for all half-edges.
- `next(prev(h)) === h` and `prev(next(h)) === h`.
- Walking `next` from any half-edge returns to it in finitely many steps, and all half-edges on that walk share the same `face` value.
- `origin(next(h)) === origin(twin(h))` (loops are vertex-continuous).
- Every face's loop has length ≥ 3.
- No two vertices within snap tolerance of each other.

This data model is the contract between the generator, the editor, and the exporter. All three operate on the same graph structure.

---

## Generation Algorithm

### Concept

Generation is an **advancing front** method. It begins with a seed polygon centered at the origin, then repeatedly grows outward by attaching new polygons to open (frontier) edges. Polygon size grows with distance from the origin, controlled by a vertex density function. The frontier is not a separate data structure — it is exactly the set of half-edges with `face: null`, organized into one or more closed boundary loops by the DCEL itself.

Because randomly sampled polygons will collide with existing geometry, and because independently growing arms of the frontier will eventually meet, generation is structured as **propose → validate → repair**: every candidate polygon is tested against the existing graph before insertion, and gaps created by merging fronts are detected and closed explicitly.

### Density Function

Vertex density decreases with radial distance `r` from the origin:

```
density(r) = D_initial / (1 + k * r)
```

- `D_initial` is the vertex density at the origin (user-specified).
- The target edge length at radius `r` is `L(r) = 1 / density(r)`.
- `k` is derived from the user-specified initial/final density ratio `ρ = D_initial / D_final` and the outer radius `R_N` of the final tier: since `density(R_N) = D_initial / (1 + k·R_N) = D_initial / ρ`, we get `k = (ρ − 1) / R_N`.

This gives a smooth, continuous size gradient with no jumps between tiers.

### Tiers

Tiers are a semantic categorization of polygons, not a structural boundary in generation.

- The user specifies the number of tiers `N`, the initial tier radial width `W_0`, and a scale factor `S`.
- Tier `i` occupies the radial band `[R_i, R_{i+1})`, where `R_0 = 0` and `R_{i+1} = R_i + W_0 * S^i`. The outer radius is `R_N`.
- A polygon's tier is the tier band containing its centroid's distance from the origin (clamped to `[0, N−1]`).
- Generation stops proposing new polygons from a frontier edge once every candidate would have a centroid beyond `R_N`; the closure pass (below) then finishes the tessellation.

### Polygon Type Distribution

The user specifies a distribution table mapping side count to probability:

```
[ { sides: 3, probability: 0.66 }, { sides: 5, probability: 0.34 }, ... ]
```

Probabilities must sum to 1.0 (UI enforces with a live validation indicator). Side counts ≥ 3 are valid; below 3 are rejected.

**The distribution is a bias, not a guarantee.** Collision repair and gap closure fill space with whatever side counts the local geometry permits, so the achieved distribution will deviate from the target. After generation, compute and display the achieved distribution next to the target.

### Spatial Index

Maintain a uniform grid hash over document space containing all vertices and all edges (as segments). Cell size ≈ the current frontier's typical edge length (rebuild or re-bucket lazily as `L(r)` grows). All proximity queries below (vertex snapping, intersection tests, point-in-face tests) go through this index; nothing scans the full graph.

### Seed

1. Sample the distribution table for a side count `n`.
2. Construct a regular `n`-gon centered at the origin with edge length `L(0) = 1 / D_initial`, at a random rotation.
3. Apply irregularity perturbation to each vertex (see step 4 below).
4. Insert it as the first face. All of its edges are frontier edges.

### Polygon Construction from a Frontier Edge

Maintain a FIFO queue of frontier half-edges. Pop a half-edge `h` with base vertices `v0 = origin(h)` and `v1 = origin(twin(h))`:

1. If `h` no longer has `face: null` (it was consumed by a snap or closure since being enqueued), skip it.
2. Sample the distribution table for a target side count `n`.
3. Estimate the candidate centroid distance `r_c` (base edge midpoint distance plus roughly the apothem of a regular `n`-gon with the local edge length) and derive the target edge length `L = 1 / density(r_c)`.
4. Place `n − 2` candidate vertices on the open (`face: null`) side of the base edge:
   - Compute idealized regular-polygon positions using edge length `L`.
   - Perturb each by a random offset scaled by `irregularity * L * 0.3` (0 = regular, 1 = maximum irregularity).
   - **Snap**: for each candidate vertex, query the spatial index for existing vertices within `0.4 * L`. If one exists, snap the candidate to it. Snapping to frontier vertices is how independently growing fronts join instead of colliding.
5. **Validate** the candidate polygon (base edge + new/snapped vertices):
   - It is a simple polygon (no self-intersection).
   - None of its edges (excluding the base edge and any edges that already exist between consecutive snapped vertices) intersects any existing edge, except at shared endpoints. (Spatial index query.)
   - No existing vertex lies strictly inside the candidate polygon.
   - Every snapped-to vertex is a frontier vertex (lies on a boundary loop); snapping to an interior vertex invalidates the candidate.
   - It passes the 1:3 aspect ratio constraint (below).
6. **On failure, repair in this order:**
   a. Re-sample vertex perturbations (up to 3 attempts at the same `n`).
   b. Reduce the side count by 1 and retry (down to `n = 3`).
   c. For an aspect-ratio failure only: nudge offending vertices inward along the polygon's longest axis until the constraint passes, then re-run the intersection tests.
   d. If even a triangle fails validation, the region around `h` is too constrained for local insertion. Leave `h` in place and mark it deferred; deferred edges are resolved by gap closure, not by re-queueing (this guarantees termination).
7. On success, insert the face: create half-edge pairs for each new edge, reuse the existing half-edges for the base edge and any pre-existing edges between snapped vertices, and fix up `next`/`prev` on the affected boundary loops.
8. Enqueue the new face's half-edges that still have `face: null` twins, unless their centroid-side test says the face they would spawn lies beyond `R_N` — in that case leave them for the closure pass.

### Gap Detection and Closure (During Generation)

When snapping joins two fronts, a boundary loop splits into two loops, one of which may be a small enclosed gap. After every insertion that involved snapping:

- Walk the boundary loop(s) touching the new face. If a loop encloses finite area (it is a hole, not the outer perimeter — distinguishable by signed area sign) and has ≤ 8 edges or area < `2 * L(r)²`, close it immediately:
  - If it has 3 edges: insert it directly as a face.
  - Otherwise: triangulate the loop polygon by ear clipping, insert the triangles, then greedily merge adjacent triangles into quads/pentagons where the merged polygon is simple and passes the aspect-ratio constraint, biased toward under-represented entries in the distribution table.
- Deferred edges (step 6d) that end up on a closed gap loop are resolved by this same mechanism.

### Final Closure Pass

When the frontier queue is exhausted:

1. Identify the outer perimeter loop: the boundary loop enclosing all others (largest absolute signed area).
2. Every other remaining boundary loop is an interior gap. Close each with the ear-clip-then-merge procedure above, regardless of size.
3. After this pass, the only `face: null` half-edges in the graph form exactly one loop: the outer perimeter. The perimeter may be ragged; that is acceptable and is the edge of the tessellation.

Closure-pass polygons are exempt from the hard aspect-ratio constraint (it is applied best-effort during triangle merging). Their side counts are whatever the geometry permits.

### Aspect Ratio Constraint

A polygon passes the 1:3 constraint if the ratio of the shorter to the longer side of its minimum-area bounding rectangle is at least 1/3. Compute the bounding rectangle by **brute-force rotation search**: rotate the polygon's convex hull through 0°–179° at 1° increments and take the orientation minimizing bounding-box area. (This is an approximation of the exact rotating-calipers result; the 1° resolution is more than sufficient here.)

### Radial Elongation (Final Modifier)

After closure and before black-fill assignment, every vertex is displaced along its radius line by the constant-anisotropy power map

    r′ = R_N · (r / R_N)^E        (angle unchanged)

with the elongation factor `E` clamped to `[0.1, 10]`; `E = 1` is the identity and the perimeter is pinned at `R_N`. This map is not a design choice among alternatives — it is forced. Under the constraint that vertices move only along their radius lines, the map must have the form `r′ = f(r)`, and its local anisotropy (radial stretch `f′(r)` over tangential stretch `f(r)/r`) is `r·f′/f`. Requiring that ratio to be the same constant `E` at every radius — so that a diamond thins into a shard by the same proportion wherever it sits — gives `f = C·r^E` uniquely. `E > 1` produces radial shards and contracts interior radii; `E < 1` produces flattened arc-like cells and pushes the interior outward.

Consequences and guards:

The per-vertex scale `(r/R_N)^{E-1}` is clamped to `[0.05, 6]`; inside the clamp the map degrades to a uniform, distortion-free scale. Without the floor, large `E` crushes the center into coincident vertices; without the cap, small `E` flings it outward without bound. The cap is the tighter bound because tangential blow-up stretches cells into long chords whose straight-line images sag across the curved map, which is the dominant failure source.

Because the map is applied to vertices while edges stay straight, extreme factors can invert a rare high-aspect face or make a long tangential chord sag radially (by `~L²/8r`) into a thinner neighbor. A repair pipeline runs after the transform: (1) inverted faces are fixed by a closed-form vertex move — the shoelace area is affine in any one vertex's position — validated against the local neighborhood, trying radial directions first and steepest-ascent second; (2) crossing pairs are separated radially by the computed sagitta over bounded passes; (3) the whole nudge phase is transactional and rolls back if it worsens the global conflict count; (4) any face still inverted is merged into a neighbor whose union is a valid simple polygon (or removed as a last resort, leaving a small reported hole). Winding validity and the single-boundary-loop property are hard guarantees. Residual edge crossings are a soft residual: counted in the generation stats and surfaced as a UI warning (empirically ~2 runs in 105 at the extremes, 1–2 crossings each). Merged or removed sliver counts are also reported.

Applied at generation time only; not re-applied during editing.

### Black Fill Assignment

After generation completes:

1. Target count: `floor(total_faces * black_probability)`.
2. Build the face adjacency graph (faces adjacent iff they share ≥ 1 edge) by iterating half-edges.
3. Greedy independent set: shuffle the face list; iterate, marking a face black if no neighbor is already black; stop at the target count.
4. **No two black faces share an edge — hard constraint.** If the greedy pass exhausts the face list before reaching the target (possible at high `black_probability`), accept the shortfall and report the achieved count and fraction to the user. Do not violate the adjacency constraint to hit the target.

---

## User-Configurable Parameters

Presented in a settings panel before generation. All parameters must be validated before the Generate button is enabled.

| Parameter | Type | Description |
|---|---|---|
| Number of tiers | integer ≥ 1 | Semantic tier count |
| Initial tier radial width | number > 0 | Width of tier 0 in document units |
| Tier scale factor | number > 0 | Multiplier on radial width per tier |
| Initial vertex density | number > 0 | Vertex density at origin |
| Initial/final density ratio | number > 1 | Controls density falloff steepness |
| Polygon irregularity | float [0, 1] | 0 = regular polygons, 1 = maximum irregularity |
| Polygon type distribution | table | Sides → probability, must sum to 1.0; treated as a bias (see above) |
| Black fill probability | float [0, 1] | Target fraction of black-filled polygons (best-effort under adjacency constraint) |
| Radial elongation | float [0.1, 10] | Final modifier: constant-anisotropy radial map; every cell's radial aspect changes by exactly this factor. 1 = off; >1 shards, <1 arcs |
| Stroke width | number > 0 | Outline thickness in document units |

Snap tolerance is not user-facing; it is derived from the local target edge length (`0.4 * L(r)`).

---

## Editor

### Canvas

- The canvas renders the DCEL graph as an SVG viewport.
- Pan: click and drag on empty canvas space.
- Zoom: scroll wheel, centered on the cursor position.
- The canvas coordinate space is document space (same as export space). No separate screen/document transform — use an SVG `viewBox` that updates on pan/zoom.

### Selection

- Click a polygon interior → select face.
- Click a vertex → select vertex.
- Click an edge → select edge (the undirected edge, i.e. a half-edge pair).
- Click empty space → deselect.
- Selected elements are highlighted visually (e.g. blue outline or handle).
- Only one element can be selected at a time; selecting a vertex deselects any selected face, etc.

### Vertex Operations

**Drag vertex:**
- Click and drag any vertex to reposition it.
- All incident edges and faces update immediately.
- No geometric constraints are enforced during drag (user editing overrides generation rules). Topology is unchanged by a drag.

**Add vertex to edge:**
- Select an edge, then choose "Add vertex" (toolbar or context menu).
- A new vertex is inserted at the edge midpoint. Both half-edges of the edge are split (producing two half-edge pairs), and `next`/`prev` are relinked. Faces on both sides (if present) gain the vertex automatically via their loops.

**Delete vertex:**
- Select a vertex, then choose "Delete vertex".
- Permitted only for degree-2 vertices. The two incident edges merge into one (splice the half-edge loops, delete the vertex and one half-edge pair).
- If merging leaves an incident face with fewer than 3 edges, that face is deleted and merged into the adjacent neighbor sharing its remaining long edge (choose the neighbor with the fewest sides if there is a choice).
- Deleting a junction vertex (degree ≥ 3) is blocked with an explanation; the user must simplify the topology first.

### Edge Operations

**Add edge:**
- Select two existing vertices (shift-click the second), then choose "Add edge".
- If both vertices lie on the same face and the open segment between them lies **strictly inside that face's interior** (it must not intersect the face boundary except at its endpoints, and must not exit the interior — check both, since the face may be concave): split the face into two faces along the new edge. Both resulting faces inherit the original fill; tiers are recomputed from centroids.
- If the segment fails the interior test, or intersects any existing edge, reject with an error message.
- If the vertices lie on boundary loops (not interior vertices) and the segment intersects nothing: add the edge as a **naked edge** (`face: null` on both sides). This is the one operation that creates naked edges; the user is responsible for continuing until the topology closes (e.g. adding further edges to bound a region — closing a cycle of naked/boundary edges around an empty region creates a new face there, filled white).

**Delete edge:**
- Select an edge, then choose "Delete edge".
- **Edge shared by two faces:** the two faces merge into their union polygon (possibly concave).
  - If the two faces share **multiple edges forming one contiguous chain**, the entire chain is removed; interior vertices of the chain (degree 2 after removal) are deleted.
  - If the two faces share **non-contiguous edges**, deletion is blocked with an explanation — the merge would produce a face with a hole, violating the simple-polygon invariant.
- **Boundary edge (one face):** the incident face is deleted along with the edge (identical to Delete Face on that face, plus removal of the selected edge if it would otherwise become naked). A confirmation is shown since this deletes a face. Rationale: a face missing one boundary edge is not a renderable simple polygon, so "keep the face open" is not a representable state.
- **Naked edge (no faces):** the edge is removed outright; endpoint vertices that drop to degree 0 are removed.

### Face Operations

**Delete face:**
- Select a face, then choose "Delete face".
- Edges shared with neighbors become boundary edges of those neighbors.
- Edges that were already boundary edges become naked edges and are removed; vertices that drop to degree 0 are removed.
- The result is a hole in the tessellation, which the user may close manually.
- Always permitted regardless of neighbor count.

### Color Operations

**Paint face:**
- Select a face, then open the color picker. Any color is permitted; stored as a hex string in `fill`.
- Painting does not enforce the no-adjacent-black constraint (user editing overrides generation rules).

**Toggle black fill:**
- Select a face, toolbar toggle. Sets fill to `#000000`. No adjacency constraint enforced during manual editing.

### Undo / Redo

- Full undo/redo covering all editing operations (vertex drag, add/delete vertex, add/delete edge, delete face, paint). A vertex drag is one operation (recorded on mouse-up).
- Shortcuts: Ctrl+Z / Ctrl+Y (Cmd+Z / Cmd+Shift+Z on Mac).
- Minimum depth: 50 operations.
- Generation is not undoable — re-generation replaces the graph entirely (confirmation dialog if unsaved edits exist).

---

## SVG Export

Export produces a clean, design-tool-ready SVG file.

### Requirements

- Every face is a separate `<path>` with absolute `M`, `L`, `Z` commands, traversed via the face's half-edge loop. No relative coordinates.
- No `transform` attributes anywhere. All coordinates in final document space.
- No `clipPath` elements.
- Faces grouped by tier: `<g id="tier-0">`, `<g id="tier-1">`, etc.
- `fill` attribute per path: white `#ffffff`, black `#000000`, custom hex as painted.
- `stroke="#000000"` and `stroke-width="<user value>"` per path (not on the group).
- **Naked edges** (zero-face edges left by manual editing) are exported as `<line>` elements with the same stroke settings, in `<g id="naked-edges">` (omit the group if empty). One-face boundary edges are **not** separately exported — they are already stroked by their face's path.
- `viewBox` is the tight bounding box of all geometry plus padding equal to the stroke width on all sides.
- No background rectangle; transparent background.
- **Winding normalization:** for each path, compute the signed shoelace sum `Σ (x_i · y_{i+1} − x_{i+1} · y_i)` over the exported coordinate sequence (SVG y-down space). If the sum is negative, reverse the coordinate order before writing. All exported paths therefore wind clockwise as seen on screen.
- Round all coordinate values to 4 decimal places.

### Export Trigger

- A clearly labeled "Export SVG" toolbar button triggers a browser download. Default filename: `tessellation.svg`.

---

## UI Layout

```
┌─────────────────────────────────────────────────────┐
│  Toolbar: [Generate] [Undo] [Redo] [Export SVG]     │
│           [Add Vertex] [Delete] [Add Edge] [Paint]  │
├──────────────┬──────────────────────────────────────┤
│              │                                      │
│  Settings    │         Canvas (SVG viewport)        │
│  Panel       │                                      │
│              │                                      │
│  (pre-gen    │                                      │
│   params)    │                                      │
│              │                                      │
└──────────────┴──────────────────────────────────────┘
```

- Settings panel visible before generation, collapsible after. After generation it also shows the achieved polygon-type distribution and achieved black-fill fraction.
- Toolbar always visible; canvas fills remaining space.
- Color picker appears as a floating panel when Paint mode is active and a face is selected.
- Right-click context menu on selected elements mirrors the toolbar actions valid for that element type.

---

## Implementation Notes for Fable

- TypeScript throughout.
- The half-edge DCEL is the single source of truth. The SVG canvas is a rendering of the graph — never modify the SVG directly and sync back.
- Render by walking face loops and emitting SVG elements; re-render on every graph mutation.
- Wrap all DCEL mutations in a small set of audited primitives (split edge, splice loops, add face to boundary cycle, remove face, merge faces). Every editor and generator operation composes these primitives; nothing touches `next`/`prev` pointers outside them. This is where half-edge implementations live or die.
- `d3-delaunay` (CDN or npm) is available as a utility; ear clipping for gap closure is simple enough to hand-roll, but constrained triangulation via d3-delaunay is an acceptable alternative for large gaps.
- The generation algorithm and editor operate on the same graph structure — no separate "generation representation."
- All IDs are UUIDs generated at creation time.
- Undo stack stores graph snapshots (deep copies). If snapshots prove slow, move to command-pattern deltas built on the mutation primitives above.
- Test generation independently before wiring up the UI. Generated output must pass:
  - All structural invariants listed in the Data Model section (twin/next/prev consistency, closed loops, per-loop face consistency).
  - Exactly one `face: null` boundary loop (the outer perimeter); no interior gaps; no naked edges.
  - No two vertices within snap tolerance of each other.
  - No two edges intersect except at shared endpoints (verifiable via the spatial index).
  - No faces with fewer than 3 edges.
- Seed the RNG explicitly and expose the seed (even if only in a debug console) so generation bugs are reproducible.
- The export function must be independently testable: given a valid graph, it produces a valid SVG string.
- Do not use `localStorage`, `sessionStorage`, or any browser storage. All state lives in memory.

---

## Out of Scope

- Multi-user or collaborative editing
- Layers
- Undo of the generation step itself
- Import of existing SVG tessellations
- Animation or transition effects
- Touch/mobile input (desktop browser only)
- Guaranteeing the exact polygon-type distribution or exact black-fill fraction (both are best-effort by design)
