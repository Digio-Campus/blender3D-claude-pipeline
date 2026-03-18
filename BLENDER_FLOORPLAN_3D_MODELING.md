# SKILL: 3D Modeling Architectural Floor Plans in Blender
> Applicable to any 2D top-down floor plan.
> Designed to be followed by an AI agent using the Blender MCP tool.
> **Phase 4 of the pipeline.** Requires handoffs from Phase 1 (wall extraction),
> Phase 2 (window & door extraction), and Phase 3 (inner wall detection).

---

## OVERVIEW OF THE FULL PIPELINE

```
Phase 1 — Outer Wall Extraction         (FLOOR_PLAN_WALL_EXTRACTION.md)
               ↓
Phase 2 — Window & Door Extraction      (WINDOW_EXTRACTION.md)
               ↓
Phase 3 — Inner Wall Detection          (FLOOR_PLAN_INNER_WALL_DETECTION.md)
               ↓
Phase 4 — 3D Build                      ← THIS SKILL
               ↓
Phase 5 — Roof Extraction & Build       (ROOF_EXTRACTION_AND_BUILD.md)
```

**This skill receives three handoff YAML blocks:**
- Phase 1: outer wall perimeter (pixel coords + scale + wall_thickness_m)
- Phase 2: windows and outer doors (pixel coords + wall assignments)
- Phase 3: inner walls and interior doors (pixel coords + thickness)

All three must be loaded before starting. Do not proceed if any handoff is missing.

---

## CRITICAL BLENDER COORDINATE SYSTEM RULES

Before writing any code, internalize these:

1. **Z-axis is UP** in Blender. Walls grow along +Z.
2. **Plan Y-axis is INVERTED**: floor plan Y increases downward, Blender Y increases upward.
   → Always negate Y when converting plan coordinates: `blender_y = -plan_y`
3. **CCW winding** (counter-clockwise) is required for correct face normals.
   → Always check with `signed_area()` and reverse if needed.
4. **Never use `bpy.ops` for geometry** — use `bmesh` directly. It's faster and more reliable.
5. **Always recalculate normals** after building geometry:
   `bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])`

---

## STEP 1 — Prepare Coordinates

> **Input convention:** All pixel coordinates from `FLOOR_PLAN_WALL_EXTRACTION.md` are **outer face** coordinates — the exterior boundary of the building. The polygon IS the outer surface of the walls. Do not treat it as a centreline.

Convert pixel perimeter to real-world meters, then to Blender coordinates:

```python
# From FLOOR_PLAN_WALL_EXTRACTION.md output:
perimeter_px = [(209,98), (378,98), ...]  # outer face pixel coords
scale_m_per_px = 9.7 / 491               # real height / pixel height
wall_thickness_m = 0.62                  # from extraction output — NEVER hardcode or guess

# Origin at top-left outer corner of building
origin_px = perimeter_px[0]

# Convert to meters (plan space, Y downward)
perimeter_m = [
    ((px - origin_px[0]) * scale_m_per_px,
     (py - origin_px[1]) * scale_m_per_px)
    for px, py in perimeter_px
]

# Convert to Blender space (Y inverted)
pts = [(x, -y) for x, y in perimeter_m]
# pts is now the OUTER FACE polygon in Blender coordinates
```

**Check winding — always:**
```python
def signed_area(pts):
    n = len(pts)
    a = 0
    for i in range(n):
        j = (i+1) % n
        a += pts[i][0]*pts[j][1] - pts[j][0]*pts[i][1]
    return a / 2

if signed_area(pts) < 0:
    pts = pts[::-1]  # ensure CCW
```

---

## STEP 2 — Compute Inner and Outer Perimeters

### THE WALL THICKNESS RULE
The input polygon `pts` **is already the outer face** of the walls (from the extraction step). Therefore:
- **Outer loop = `pts` directly** — no offset needed.
- **Inner loop = `pts` offset inward by the full wall thickness `T`**.

`T` must come from `wall_thickness_m` in the extraction output. **Never hardcode it.**

```python
T = wall_thickness_m  # e.g. 0.62 — always read from extraction output
```

**DO NOT use the Solidify modifier** for non-convex (L-shaped, U-shaped, notched) floor plans. It produces corner spikes and stretched geometry at re-entrant (concave) corners. **Build inner and outer loops manually instead.**

### For strictly 90° polygons (all architectural plans):

```python
import numpy as np

def axis_offset_90deg(pts, i, h):
    """
    For a 90° corner, offset inward by h meters.
    h = 0   → outer face (no offset) — use this for the outer loop
    h = T   → inner face (full thickness inward) — use this for the inner loop
    h > 0 always means inward for CCW winding.
    """
    n = len(pts)
    prev = np.array(pts[(i-1)%n])
    curr = np.array(pts[i])
    nxt  = np.array(pts[(i+1)%n])

    e1 = curr - prev; e1 = e1/np.linalg.norm(e1) if np.linalg.norm(e1)>1e-10 else e1
    e2 = nxt - curr;  e2 = e2/np.linalg.norm(e2) if np.linalg.norm(e2)>1e-10 else e2

    # Left normal = inward for CCW winding
    n1 = np.array([-e1[1],  e1[0]])
    n2 = np.array([-e2[1],  e2[0]])
    inward = n1 + n2
    il = np.linalg.norm(inward)
    if il < 1e-10:
        return (curr[0], curr[1])  # degenerate — handle separately (see below)

    offset = h * inward / il * np.sqrt(2)  # sqrt(2) corrects for 45° bisector at 90° corners
    return (round(curr[0] + offset[0], 5), round(curr[1] + offset[1], 5))

# Outer loop = the pts polygon itself (outer face — no offset)
outer = list(pts)

# Inner loop = pts offset inward by full wall thickness T
inner = [axis_offset_90deg(pts, i, T) for i in range(len(pts))]
```

### CRITICAL: Re-entrant (Concave) Corner Handling

A re-entrant corner is where the perimeter doubles back — incoming and outgoing edges are **parallel and opposite** (180° turn). The offset formula returns zero at these points, causing spikes on the inner loop.

**The outer loop is unaffected** — it is just `pts` with no computation. Only the inner loop needs special treatment at re-entrant corners.

**How to detect:**
```python
# incoming edge direction == opposite of outgoing edge direction
# e.g. incoming = RIGHT (+X), outgoing = LEFT (-X) → 180° turn
```

**How to fix — split into TWO inner vertices:**

At a re-entrant corner the inner loop must "wrap around" the inside of the corner. Since the outer loop stays at the exact corner pixel, the inner loop needs two points to turn the 90° inside corner:

```python
# At re-entrant corner index i with outer face point (cx, cy):
# Incoming edge goes in direction d_in, outgoing in direction d_out = -d_in
#
# Example: incoming goes RIGHT (+X), outgoing goes LEFT (-X), corner at (cx, cy)
# The inner loop wraps around the inside of the notch tip:
inner_a = (cx + T, cy + T)  # end of incoming wall inner face
inner_b = (cx + T, cy - T)  # start of outgoing wall inner face
# (signs depend on which direction the re-entrant corner faces — see below)

# Insert these TWO inner points in place of the single degenerate point.
# The outer loop keeps its single point at (cx, cy) unchanged.
# outer loop: [..., (cx, cy), ...]           ← unchanged, single point
# inner loop: [..., inner_a, inner_b, ...]   ← two points replace one
```

**Sign convention depends on which direction the re-entrant corner faces:**
- Corner on RIGHT outer wall (x = max): inner_a X = cx − T, wrap in Y
- Corner on LEFT outer wall (x = min):  inner_a X = cx + T, wrap in Y
- Corner on TOP outer wall (y = max):   inner_a Y = cy − T, wrap in X
- Corner on BOTTOM outer wall (y = min): inner_a Y = cy + T, wrap in X

After splitting, `len(inner)` will be `len(outer) + 1` per re-entrant corner — this is correct. The mesh-building loop in Step 3 must account for this by iterating over `inner` pairs independently from `outer` if counts differ, or by inserting a matching duplicate point in the outer loop at each split position to keep lengths equal.

---

## STEP 3 — Build the Mesh in Blender

```python
import bpy
import bmesh

WALL_HEIGHT = 2.70  # standard European ceiling height (meters)

mesh = bpy.data.meshes.new("OuterWalls")
obj  = bpy.data.objects.new("OuterWalls", mesh)
bpy.context.collection.objects.link(obj)
bpy.context.view_layer.objects.active = obj
obj.select_set(True)

bm = bmesh.new()

n = len(outer)  # outer and inner must have same length

# Create vertex rings
ob  = [bm.verts.new((x, y, 0.0))          for x,y in outer]  # outer bottom
ot  = [bm.verts.new((x, y, WALL_HEIGHT))  for x,y in outer]  # outer top
ib  = [bm.verts.new((x, y, 0.0))          for x,y in inner]  # inner bottom
it_ = [bm.verts.new((x, y, WALL_HEIGHT))  for x,y in inner]  # inner top
bm.verts.ensure_lookup_table()

# Build 4 face loops
for i in range(n):
    j = (i+1) % n
    bm.faces.new([ob[i],  ob[j],  ot[j],  ot[i]])   # outer vertical face
    bm.faces.new([ib[j],  ib[i],  it_[i], it_[j]])  # inner vertical face (flipped)
    bm.faces.new([ot[i],  ot[j],  it_[j], it_[i]])  # top cap
    bm.faces.new([ob[j],  ob[i],  ib[i],  ib[j]])   # bottom cap

# ALWAYS recalculate normals after building
bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])

bm.to_mesh(mesh)
bm.free()
mesh.update()
```

### Why NOT Solidify:
| Situation | Solidify result | Manual loop result |
|---|---|---|
| Simple convex rectangle | ✅ Clean | ✅ Clean |
| L-shape or U-shape | ⚠️ Corner spikes | ✅ Clean |
| Re-entrant corner | ❌ Stretched/exploded | ✅ Clean |
| Complex floor plan | ❌ Unreliable | ✅ Always correct |

---

## STEP 4 — Verify After Each Build

**Always take a viewport screenshot after building geometry:**

```python
# Top-down orthographic view
import mathutils
for area in bpy.context.screen.areas:
    if area.type == 'VIEW_3D':
        for space in area.spaces:
            if space.type == 'VIEW_3D':
                space.region_3d.view_perspective = 'ORTHO'
                space.region_3d.view_rotation = mathutils.Euler((0,0,0),'XYZ').to_quaternion()
                space.region_3d.view_distance = 22
                space.region_3d.view_location = (cx, cy, 0)  # center of building
        break
```

**Checklist after each wall build:**
- [ ] All corners are strictly 90° (no bevels, no spikes, no stretched faces)
- [ ] Wall thickness is uniform along all segments
- [ ] No faces are missing (no holes in the mesh)
- [ ] Top and bottom caps are present
- [ ] Model dimensions match real-world measurements

**Quick dimension check:**
```python
obj = bpy.data.objects['OuterWalls']
print(f"Dimensions: {obj.dimensions}")  # should match real-world building size
```

---

## STEP 5 — Cut Window and Door Openings

Load the Phase 2 handoff and cut all openings into the OuterWalls mesh using
Boolean DIFFERENCE operations.

### 5a — Load Phase 2 handoff

```python
# Phase 2 handoff provides, for each opening:
#   name, wall, px coords, and (for windows) sill_height + opening_height
#   (for doors) opening_height only (sill = 0)

SILL_H   = 0.90   # m — standard European window sill
WIN_H    = 1.20   # m — standard European window height
DOOR_H   = 2.10   # m — standard European door height
WALL_T   = wall_thickness_m + 0.10  # cutter slightly thicker than wall

def px_to_blender(col, row, scale, ox, oy):
    return round((col - ox) * scale, 4), round(-(row - oy) * scale, 4)
```

### 5b — Build cutter boxes

For each opening, create a box slightly thicker than the outer wall so the
Boolean cuts completely through both faces:

```python
import bpy

def make_opening_cutter(name, cx, cy, cz, sx, sy, sz):
    """
    cx, cy, cz = centre of the cutter box in Blender space
    sx, sy, sz = full dimensions (width × wall_depth × height)
    """
    bpy.ops.mesh.primitive_cube_add(location=(cx, cy, cz))
    obj = bpy.context.object
    obj.name = f"CUT_{name}"
    obj.scale = (sx/2, sy/2, sz/2)
    bpy.ops.object.transform_apply(scale=True)
    return obj

cutters = []

# Example — horizontal wall window (wall parallel to X axis):
# col_left, col_right = pixel bounds of opening
# wall_y_blender = Y position of wall centreline in Blender space
for opening in phase2_openings:
    x1, _ = px_to_blender(opening['col_left'],  opening['wall_row'], SCALE, OX, OY)
    x2, _ = px_to_blender(opening['col_right'], opening['wall_row'], SCALE, OX, OY)
    cx = (x1 + x2) / 2
    cy = opening['wall_y_blender']
    sx = abs(x2 - x1)
    sy = WALL_T
    if opening['type'] == 'window':
        sz = WIN_H
        cz = SILL_H + WIN_H / 2
    else:  # door
        sz = DOOR_H
        cz = DOOR_H / 2
    cutters.append(make_opening_cutter(opening['name'], cx, cy, cz, sx, sy, sz))
```

### 5c — Apply Boolean DIFFERENCE

Join all cutters into one object, apply a single Boolean DIFFERENCE, then
delete the cutter and recalculate normals:

```python
# Join all cutters
bpy.ops.object.select_all(action='DESELECT')
for obj in cutters:
    obj.select_set(True)
bpy.context.view_layer.objects.active = cutters[0]
bpy.ops.object.join()
cutter = bpy.context.object
cutter.name = "OpeningCutters"

# Apply to OuterWalls
wall = bpy.data.objects['OuterWalls']
bpy.ops.object.select_all(action='DESELECT')
wall.select_set(True)
bpy.context.view_layer.objects.active = wall

mod = wall.modifiers.new(name="OpeningCuts", type='BOOLEAN')
mod.operation = 'DIFFERENCE'
mod.object    = cutter
mod.solver    = 'EXACT'       # ← ALWAYS use EXACT. 'FAST' does not exist in current Blender.
bpy.ops.object.modifier_apply(modifier="OpeningCuts")

# Delete cutter
bpy.ops.object.select_all(action='DESELECT')
cutter.select_set(True)
bpy.ops.object.delete()

# Recalculate normals
wall.select_set(True)
bpy.context.view_layer.objects.active = wall
bpy.ops.object.mode_set(mode='EDIT')
bpy.ops.mesh.select_all(action='SELECT')
bpy.ops.mesh.normals_make_consistent(inside=False)
bpy.ops.object.mode_set(mode='OBJECT')
```

### 5d — Verify openings from the exterior

Take a viewport screenshot from outside each wall that has openings.
**The correct Boolean solver is `'EXACT'`** — if you see an error saying
the solver name is not found, you have used `'FAST'` or `'MANIFOLD'`
by mistake.

**Visual checklist after cutting:**
- [ ] All window openings have a sill (solid wall below) and a lintel (solid wall above)
- [ ] All door openings go from Z=0 to Z=DOOR_H with no sill
- [ ] No inner wall end cap is visible from outside through any window opening
  (if visible, the inner wall endpoint is inside the outer wall cavity — see Step 6c)
- [ ] No double-cutting artefacts (stray faces inside openings)

---

## STEP 6 — Interior Walls

Interior wall geometry is detected in **Phase 3** (FLOOR_PLAN_INNER_WALL_DETECTION.md),
which produces a handoff YAML of confirmed wall segments with pixel coordinates,
thickness, and orientation. This step consumes that handoff and builds the
geometry in Blender.

**Do not re-detect inner walls here.** Load the Phase 3 handoff directly.

Each inner wall segment is a rectangular box built with `make_wall_segment`.
They do NOT need inner/outer loops — a simple extruded rectangle is correct.

```python
def make_wall_segment(bm, x0, y0, x1, y1, thickness, height):
    """
    Create a single interior wall segment between two points.
    The wall is centred on the line from (x0,y0) to (x1,y1).
    """
    import numpy as np

    p0 = np.array([x0, y0])
    p1 = np.array([x1, y1])

    direction = p1 - p0
    length = np.linalg.norm(direction)
    if length < 1e-10:
        return

    d = direction / length
    perp = np.array([-d[1], d[0]]) * (thickness / 2)

    c0 = p0 + perp;  c1 = p0 - perp
    c2 = p1 - perp;  c3 = p1 + perp

    vb = [bm.verts.new((c[0], c[1], 0.0))     for c in [c0,c1,c2,c3]]
    vt = [bm.verts.new((c[0], c[1], height))  for c in [c0,c1,c2,c3]]

    bm.faces.new([vb[0], vb[1], vb[2], vb[3]])  # bottom
    bm.faces.new([vt[3], vt[2], vt[1], vt[0]])  # top
    bm.faces.new([vb[0], vb[3], vt[3], vt[0]])  # side A
    bm.faces.new([vb[1], vb[0], vt[0], vt[1]])  # end A
    bm.faces.new([vb[2], vb[1], vt[1], vt[2]])  # side B
    bm.faces.new([vb[3], vb[2], vt[2], vt[3]])  # end B
```

**Interior wall thickness:** Load-bearing inner walls (muros de carga) may be
0.25–0.60m; standard partitions are 0.10–0.15m. Always use the value from the
Phase 3 handoff — never assume a default.

### 6a — CRITICAL: Endpoints must sit at the outer wall INNER FACE

⚠️ **This is the single most common error when building inner walls.**

When an inner wall terminates at an outer wall, its endpoint must be placed
**exactly at the inner face** of that outer wall — not at the wall centre,
not past it.

**Why:** The outer wall has a cavity of thickness T. Window openings cut through
this entire cavity. If an inner wall endpoint sits anywhere between the outer face
and the inner face, its end cap is inside that cavity and fully visible from
outside through every window on that wall.

```
WRONG — endpoint at outer wall centre:
  ─────────────────┬──────────────┬──────── (top wall)
  outer face       │  end cap     │  inner face
  (exterior)       │  VISIBLE!    │  (interior)
  ─────────────────┴──────────────┴────────
                   window cuts here

CORRECT — endpoint at inner face:
  ──────────────────────────────┬────────── (top wall)
  outer face                    │  end cap
  (exterior)                    │  flush ✓
  ──────────────────────────────┴──────────
                                inner face
```

**Formula for every endpoint that touches an outer wall:**

```python
H = wall_thickness_m / 2   # outer wall half-thickness

# Inner face positions (adjust sign for each wall's inward direction):
Y_TOP_INNER    = wall_centre_top    - H   # interior is southward (−Y)
Y_BOTTOM_INNER = wall_centre_bottom + H   # interior is northward (+Y)
X_LEFT_INNER   = wall_centre_left   + H   # interior is rightward (+X)
X_RIGHT_INNER  = wall_centre_right  - H   # interior is leftward  (−X)
# Same logic applies to all step/ledge walls.

# Set the endpoint to exactly this value — no more, no less.
```

**Validation check before finalising every endpoint:**

```python
def endpoint_in_cavity(val, outer_face, inner_face):
    """Returns True if the endpoint is inside the outer wall cavity (wrong)."""
    lo, hi = min(outer_face, inner_face), max(outer_face, inner_face)
    return lo < val < hi   # strictly between faces = in the cavity

# If True → set val = inner_face exactly and rebuild.
```

### 6b — Verify inner walls from the exterior

After building all inner walls, take a viewport screenshot from outside each
windowed wall face. No inner wall end cap should be visible through any opening.

If an end cap is visible:
1. Identify which inner wall it belongs to
2. Check its endpoint value — it has drifted to the wall centre or past it
3. Move it to exactly `inner_face` and rebuild that segment

---

## COMMON ERRORS AND FIXES

| Error | Symptom | Cause | Fix |
|---|---|---|---|
| Wrong wall thickness | Walls too thin or too thick | T hardcoded to 0.30 instead of reading `wall_thickness_m` | Always set `T = wall_thickness_m` from extraction output |
| Wall shifted inward | Outer face of 3D wall sits inside the image boundary | Treating outer face polygon as centreline and offsetting outward | `outer = list(pts)` — no outward offset; pts IS the outer face |
| Solidify spike | Wall stretches past corner | Re-entrant corner + Solidify | Use manual inner loop only; outer loop = pts |
| Inverted normals | Faces appear dark/inside-out | Wrong face winding order | `bmesh.ops.recalc_face_normals()` |
| Flat-looking walls | No height in perspective | Z not applied to top verts | Check `WALL_HEIGHT` is non-zero |
| Wrong shape | Model mirrored horizontally | Y not inverted | Apply `blender_y = -plan_y` |
| Gaps in mesh | Missing faces at corners | Loop index mismatch at re-entrant corners | Insert duplicate outer point alongside each inner split pair |
| Beveled corners | Angled instead of 90° | Miter offset used | Use `axis_offset_90deg()` function |
| Degenerate inner corner | Zero-size offset on inner loop | 180° turn in perimeter (re-entrant corner) | Split inner loop into two vertices (see Step 2); outer loop unchanged |
| Boolean solver error | `enum "FAST" not found` error in Blender | Used `mod.solver = 'FAST'` which does not exist in current Blender | Use `mod.solver = 'EXACT'` — the only valid solver for clean architectural geometry |
| Inner wall end cap visible from outside | Looking through a window reveals a flat face of an inner wall | Inner wall endpoint placed at outer wall centre or past it — inside the window cavity | Move endpoint to exactly the outer wall **inner face** coordinate. See Step 6a and `endpoint_in_cavity()` |
| Gap between inner wall and outer wall | Thin crack of light visible where inner wall meets outer wall | Endpoint placed short of the outer wall inner face | Move endpoint to exactly the inner face. Ensure `endpoint_in_cavity()` returns False and gap = 0 |
| Tried to fix a gap but end cap reappeared | Fixed a gap by pushing endpoint to wall centre — now visible through window again | Confused "inside the wall body" with "at the inner face" — the centre is too far | The only correct target is the inner face coordinate, not the centre |

---

## WORKFLOW SUMMARY

```
0. Load Phase 1 handoff → perimeter_px, scale, wall_thickness_m
   Load Phase 2 handoff → windows and outer doors (pixel bounds + wall assignments)
   Load Phase 3 handoff → inner walls and interior doors (pixel bounds + thickness)

OUTER WALLS:
1. Set T = wall_thickness_m — NEVER hardcode T
2. Convert pixels → metres → Blender coords (negate Y)
3. Ensure CCW winding (pts is the outer face polygon)
4. outer loop = pts  (no offset — already the outer face)
5. Identify re-entrant corners (180° turns in pts) — split inner loop into 2 verts at each
6. inner loop = pts offset inward by full T using axis_offset_90deg(pts, i, T)
7. Build mesh with 4 face loops: outer vertical, inner vertical, top cap, bottom cap
8. Recalculate normals
9. Verify top-down: wall shape matches plan

OPENINGS (Phase 2):
10. Build cutter boxes for each window (sill=0.90m, h=1.20m) and door (sill=0, h=2.10m)
    Each cutter thickness = wall_thickness_m + 0.10m (slightly proud of both faces)
11. Join all cutters → apply Boolean DIFFERENCE with solver='EXACT'
12. Delete cutters, recalculate normals
13. Verify from exterior: clean openings, no artefacts

INNER WALLS (Phase 3):
14. For each inner wall segment from Phase 3 handoff, call make_wall_segment()
15. For every endpoint that touches an outer wall:
    - Set coordinate to exactly the outer wall INNER FACE (not centre, not past it)
    - Run endpoint_in_cavity() check — must return False
16. Build all segments into a single InnerWalls mesh
17. Recalculate normals
18. Verify from exterior through each window: no inner wall end caps visible
```

---

## ARCHITECTURAL DEFAULTS (European)

| Element | Default Value |
|---|---|
| Exterior wall thickness | 0.30 m |
| Interior wall thickness | 0.10–0.15 m |
| Ceiling / wall height | 2.70 m |
| Door opening width | 0.90 m |
| Door opening height | 2.10 m |
| Window sill height | 0.90 m |
| Window height | 1.20 m |

---

## NOTES ON THE BACKGROUND PLAN IMAGE

If a background floor plan image is present in Blender as an EMPTY object:
- Its scale in Blender units does NOT necessarily match real-world meters
- **Do NOT rescale your geometry to match the background image**
- The background is a visual reference only — trust the real-world meter coordinates
- Use top-down orthographic view to visually verify the shape matches the plan layout
- If alignment is needed, move the model to match the image, not the other way around
