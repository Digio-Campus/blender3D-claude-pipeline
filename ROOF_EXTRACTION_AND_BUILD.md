# SKILL: Roof Extraction and 3D Build from Architectural Roof Plans
> Applicable to any 2D top-down architectural roof plan image.
> Designed to be followed by an AI agent using PIL/numpy for pixel analysis
> and the Blender MCP tool for 3D construction.
> **Phase 5 of the pipeline. Requires handoffs from Phase 1 (outer wall geometry)
> and Phase 4 (3D outer wall model already built in Blender).**

---

## OVERVIEW

This is **Phase 5** of the floor plan pipeline. It receives the outer wall
perimeter from Phase 1 and the existing 3D model from Phase 3, then:

1. Extracts the roof geometry from the roof plan image
2. Identifies roof type, ridge lines, slopes, overhangs, and drainage
3. Builds the roof as a 3D mesh in Blender, sitting on top of the outer walls

A roof plan is a **separate drawing** from the floor plan — it is a top-down
view of the building looking down at the roof surface, typically showing:
- The outer wall perimeter (as a thin rectangle with dashed or dotted lines)
- Ridge lines (the apex lines where two roof faces meet)
- Hip lines (diagonal lines at corners where two sloped faces meet)
- Overhang lines (the outer edge of the roof, beyond the outer walls)
- Slope direction arrows or percentage/degree annotations
- Drainage gutters or downpipe symbols (circles with a cross)

**Do NOT attempt to derive roof geometry from the floor plan.** The floor plan
contains no reliable roof information. Always require a roof plan before
proceeding.

---

## PIPELINE POSITION

```
Phase 1 — Outer Wall Extraction         (FLOOR_PLAN_WALL_EXTRACTION.md)
Phase 2 — Window & Door Extraction      (WINDOW_EXTRACTION.md)
Phase 3 — Inner Wall Detection          (FLOOR_PLAN_INNER_WALL_DETECTION.md)
Phase 4 — 3D Outer Walls + Openings     (BLENDER_FLOORPLAN_3D_MODELING.md)
Phase 5 — Roof Extraction & Build       ← THIS SKILL
```

**Phase 5 inputs:**
- The roof plan image
- Phase 1 handoff: `scale`, `origin`, outer wall perimeter in metres
- The Blender scene from Phase 3 (OuterWalls object must exist)

---

## CRITICAL WARNINGS — READ BEFORE STARTING

### 1. A roof plan is NOT a floor plan
The roof plan is drawn at the same scale but shows the building from a
bird's-eye view of the roof surface. The outer wall perimeter appears as a
**dashed or thin dotted rectangle** (not a solid thick band), because it is
a reference projection from below, not an actual element at roof level.

**Never try to extract the roof perimeter from the floor plan.** Use the Phase 1
outer wall coordinates directly as the base of the roof.

### 2. Roof type must be identified before anything else
The roof type determines the entire geometry. Common types:

| Type | Description | Ridge | Hip lines | Gable |
|---|---|---|---|---|
| **Flat** | Horizontal plane, slight drainage slope | None | None | None |
| **Shed** (lean-to) | Single sloped plane | None | None | One |
| **Gable** | Two sloped faces meeting at a central ridge | One (horizontal) | None | Two |
| **Hip** | Four sloped faces, all edges sloped | One (horizontal) | Four diagonal | None |
| **Half-hip** (jerkinhead) | Hip at one end, gable at other | One (horizontal) | Two diagonal | One |
| **Pyramid** | Four triangular faces meeting at a point | None | Four diagonal | None |
| **Mansard** | Double slope on all four sides | None | Four | None |
| **Butterfly** | V-shape, two faces slope inward | One (valley) | None | Two |
| **Complex** | Combination of the above | Multiple | Multiple | Multiple |

**Identify the type visually first** from the roof plan before running any
pixel analysis. Write it down. All subsequent steps depend on it.

### 3. The overhang is NOT the wall perimeter
Most roofs extend beyond the outer walls — this extension is the **overhang**
(also called the eave). The outer edge of the roof visible in the roof plan is
the overhang edge, NOT the wall face. The wall perimeter sits inside it.

**Measure the overhang offset separately.** Standard European residential
overhang = 0.40–0.60m. This will need to be added outward from the Phase 1
wall perimeter coordinates when building the roof mesh.

### 4. Scale must match Phase 1 exactly
The roof plan is typically drawn at the same scale as the floor plan. Verify
this by cross-checking a known dimension (e.g. building total width) against
the Phase 1 scale. If the roof plan is at a different scale, derive a new
scale factor and keep it separate — never mix the two.

### 5. Roof height is almost never labeled
Roof ridge height is rarely shown on a 2D roof plan. It must be derived from:
- The **slope angle** (if labeled, e.g. "35°" or "1:1.5")
- The **roof pitch notation** (e.g. "4/12" = 4 units rise per 12 units run)
- The **building half-width** and the slope angle together
- Or asked from the user if not annotated

If no slope information is present, ask the user before building.

### 6. Dormer windows require the floor plan AND the roof plan together
Dormers are vertical structures that project from a sloped roof. They appear
in the roof plan as rectangles or U-shapes interrupting a slope line. Their
position along the slope must be cross-referenced with the floor plan to
determine their height. Do not attempt to model dormers without both plans.

---

## STEP 1 — Identify Roof Type and Key Lines

Load the roof plan image and visually scan it before any pixel analysis.

```python
from PIL import Image
import numpy as np

img = Image.open('roof_plan.png').convert('RGB')
arr = np.array(img)
gray = arr[:, :, 0]
```

**Visual checklist:**
- [ ] What is the roof type? (see table in Warning 2)
- [ ] Is there a single ridge line, multiple ridges, or none?
- [ ] Are there hip lines (diagonal corner lines)?
- [ ] Is the overhang edge visible as a separate outer rectangle?
- [ ] Are there slope direction arrows? Which direction do they point?
- [ ] Are there labeled dimensions (ridge height, slope angle, overhang)?
- [ ] Are there any dormers, skylights, or roof penetrations?
- [ ] Are there drainage symbols or valley lines?

Save zoomed crops of any annotated areas (slope labels, dimension lines) for
later reference.

---

## STEP 2 — Extract the Roof Outline (Overhang Edge)

The outer edge of the roof in the plan is the eave/overhang line. Extract it
using the same projection-scan technique as Phase 1, but looking for the
outermost continuous rectangle — which will be slightly larger than the wall
perimeter.

```python
# Find the outermost dark rectangle in the roof plan
# (same technique as FLOOR_PLAN_WALL_EXTRACTION.md Step 2-4)
# The roof outline is typically a SINGLE thin line (1-2px), not a double band.

def find_outermost_line(gray, axis, direction, scan_start, scan_end,
                        pos_start, pos_end, thresh=140):
    """
    Scan inward from one edge to find the first continuous dark line.
    axis: 'h' = scan rows (looking for horizontal line)
          'v' = scan cols (looking for vertical line)
    direction: +1 = scan from low index upward, -1 = scan from high index downward
    """
    positions = range(scan_start, scan_end, direction)
    for pos in positions:
        if axis == 'h':
            line = gray[pos, pos_start:pos_end]
        else:
            line = gray[pos_start:pos_end, pos]
        dark_count = np.sum(line < thresh)
        coverage = dark_count / (pos_end - pos_start)
        if coverage > 0.6:   # 60% of line is dark = a real roof edge
            return pos
    return None
```

**Cross-check:** the roof outline should be consistently larger than the Phase 1
wall perimeter by approximately the overhang distance on all sides.

---

## STEP 3 — Extract Ridge and Hip Lines

For gable, hip, and complex roofs, the ridge and hip lines define the roof
geometry. These are the internal lines inside the roof outline.

### Ridge line (horizontal apex)
For a standard gable or hip roof, the ridge runs parallel to the long axis of
the building, centred between the two eave edges it is perpendicular to.

```python
def find_ridge_line(gray, roof_outline, orientation='horizontal',
                    thresh=140, min_length=50):
    """
    Search for internal dark lines parallel to the ridge direction,
    within the roof outline bounding box.
    Returns (fixed_coord, start, end) for the ridge line.
    """
    col_min = roof_outline['left']
    col_max = roof_outline['right']
    row_min = roof_outline['top']
    row_max = roof_outline['bottom']

    candidates = []
    if orientation == 'horizontal':
        for row in range(row_min + 10, row_max - 10):
            line = gray[row, col_min:col_max]
            dark = np.sum(line < thresh)
            if dark / (col_max - col_min) > 0.4:
                candidates.append(row)
    else:
        for col in range(col_min + 10, col_max - 10):
            line = gray[row_min:row_max, col]
            dark = np.sum(line < thresh)
            if dark / (row_max - row_min) > 0.4:
                candidates.append(col)

    # Cluster and take centre
    if not candidates:
        return None
    # Simple median of all candidates
    return int(np.median(candidates))
```

### Hip lines (diagonal)
Hip lines run diagonally from the corners of the building to the ends of the
ridge. They are only present in hip, half-hip, and pyramid roofs.

**Detection approach:** hip lines appear as diagonal dark lines in the roof
plan interior. They cannot be detected by simple row/column scanning. Instead:

```python
# Use edge detection and Hough transform for diagonal lines
from PIL import ImageFilter

# 1. Extract interior region (inside roof outline)
interior = gray[roof_outline['top']:roof_outline['bottom'],
                roof_outline['left']:roof_outline['right']]

# 2. Threshold to binary
binary = (interior < thresh).astype(np.uint8) * 255

# 3. Visual inspection is more reliable than automated detection for hip lines.
# Save the interior crop at high zoom and identify hip lines manually.
interior_img = Image.fromarray(binary)
interior_img = interior_img.resize(
    (interior_img.width * 4, interior_img.height * 4), Image.NEAREST)
interior_img.save('roof_interior_zoom.png')
```

For hip lines, **visual identification is preferred** over automated detection.
After identifying the hip line endpoints visually, record them as pixel
coordinates and convert to metres.

---

## STEP 4 — Measure Slope and Derive Ridge Height

Roof slope is typically shown in one of these notations in the plan:

| Notation | Meaning | How to convert to angle |
|---|---|---|
| `35°` | Direct angle | Use directly |
| `4/12` or `4:12` | Rise/run (imperial) | `atan(4/12)` = 18.4° |
| `1:1.5` | Rise:run (metric) | `atan(1/1.5)` = 33.7° |
| `30%` | Percentage slope | `atan(0.30)` = 16.7° |
| Arrow only | Slope direction only, no angle | Ask user for angle |

```python
import math

def ridge_height_from_slope(half_span_m, slope_angle_deg):
    """
    For a symmetric gable or hip roof:
    half_span = distance from eave to ridge centreline (metres)
    Returns the height of the ridge above the wall top plate.
    """
    return half_span_m * math.tan(math.radians(slope_angle_deg))

# Example: 8m wide building, 35° slope
# half_span = 8/2 = 4.0m (eave to ridge centre)
# ridge_height = 4.0 * tan(35°) = 2.80m
```

**If no slope annotation is present:** ask the user before proceeding.
Do not assume a default slope — it significantly affects the roof appearance.

---

## STEP 5 — Verification Image

Draw the extracted roof geometry over the roof plan image before building in
Blender.

```python
from PIL import Image, ImageDraw

img_verify = Image.open('roof_plan.png').convert('RGB')
draw = ImageDraw.Draw(img_verify)

CYAN   = (0, 200, 200)    # overhang outline
RED    = (220, 0, 0)      # ridge line
ORANGE = (255, 140, 0)    # hip lines
BLUE   = (0, 80, 255)     # valley lines

# Draw overhang outline
draw.rectangle([overhang_left, overhang_top, overhang_right, overhang_bottom],
               outline=CYAN, width=3)

# Draw ridge
if ridge:
    draw.line([(ridge['x1'], ridge['y']), (ridge['x2'], ridge['y'])],
              fill=RED, width=3)

# Draw hip lines
for hip in hip_lines:
    draw.line([(hip['x1'], hip['y1']), (hip['x2'], hip['y2'])],
              fill=ORANGE, width=3)

img_verify.save('roof_verification.png')
```

**Do NOT proceed to Blender until the user has approved the verification image.**

---

## STEP 6 — Convert to Blender Coordinates

Use the Phase 1 SCALE and origin for all coordinate conversions.

```python
# Phase 1 handoff values (load from file or use stored values)
SCALE = ...    # m/px
OX, OY = ...   # origin pixel

def px_to_blender(col, row):
    x = (col - OX) * SCALE
    y = -(row - OY) * SCALE   # Y inverted
    return round(x, 4), round(y, 4)

# Overhang offset (typically measured from wall perimeter in pixels)
OVERHANG_M = ...   # e.g. 0.50m

# Wall top height (from Phase 3)
WALL_HEIGHT = 2.70   # m

# Ridge height (computed in Step 4)
RIDGE_HEIGHT = WALL_HEIGHT + ridge_height_from_slope(half_span_m, slope_angle_deg)
```

---

## STEP 7 — Build Roof in Blender

The approach depends on the roof type identified in Step 1.

### 7a. FLAT ROOF

```python
import bpy
import bmesh

def build_flat_roof(wall_perimeter_m, overhang_m, wall_height,
                    drain_slope=0.02):
    """
    Build a flat roof with a slight drainage slope.
    wall_perimeter_m: list of (x, y) points (CCW, Blender space)
    overhang_m: distance to extend outward from each wall face
    drain_slope: height difference across the roof for drainage (default 2%)
    """
    import numpy as np

    # Offset perimeter outward by overhang
    n = len(wall_perimeter_m)
    pts = [np.array(p) for p in wall_perimeter_m]

    def outward_offset(pts, i, dist):
        prev = pts[(i-1) % n]
        curr = pts[i]
        nxt  = pts[(i+1) % n]
        e1 = curr - prev; e1 = e1 / np.linalg.norm(e1)
        e2 = nxt - curr;  e2 = e2 / np.linalg.norm(e2)
        n1 = np.array([e1[1], -e1[0]])   # right-normal for CW = outward for CCW
        n2 = np.array([e2[1], -e2[0]])
        outward = n1 + n2
        ol = np.linalg.norm(outward)
        if ol < 1e-10:
            return curr
        return curr + dist * outward / ol * np.sqrt(2)

    eave_pts = [outward_offset(pts, i, overhang_m) for i in range(n)]

    mesh = bpy.data.meshes.new("Roof")
    obj  = bpy.data.objects.new("Roof", mesh)
    bpy.context.collection.objects.link(obj)

    bm = bmesh.new()
    verts = [bm.verts.new((p[0], p[1], wall_height)) for p in eave_pts]
    bm.faces.new(verts)
    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])
    bm.to_mesh(mesh)
    bm.free()
    mesh.update()
```

### 7b. GABLE ROOF

```python
def build_gable_roof(eave_pts_long_sides, ridge_y, wall_height,
                     ridge_height, overhang_m, gable_overhang_m):
    """
    Build a simple symmetric gable roof.

    eave_pts_long_sides: the two long eave edges [(x1,y1),(x2,y2)] each
    ridge_y: Y coordinate of the ridge centreline (Blender space)
    wall_height: Z of the wall top plate
    ridge_height: Z of the ridge apex
    overhang_m: eave overhang perpendicular to ridge
    gable_overhang_m: overhang parallel to ridge (at gable ends)
    """
    # The roof has 6 key vertices:
    # 2 ridge endpoints (top centre)
    # 4 eave corners (bottom edges of each slope face)

    # Ridge endpoints: at the two gable ends + gable overhang
    rx1 = eave_pts_long_sides[0][0] - gable_overhang_m
    rx2 = eave_pts_long_sides[1][0] + gable_overhang_m
    rz  = ridge_height

    # Eave corners: the four corners of the roof footprint
    # (eave_pts already include the eave overhang offset)
    # ...

    # Build two rectangular slope faces + two triangular gable faces
    pass  # implement per-project based on building orientation
```

### 7c. HIP ROOF

```python
def build_hip_roof(wall_perimeter_m, overhang_m, wall_height, ridge_height):
    """
    Build a hip roof. The ridge is centred and shorter than the building.
    All four faces are trapezoidal (or triangular for pyramid roofs).

    Key geometry: for a rectangular building W × L with equal slope on all sides:
    - Ridge length = L - W (if L > W), positioned at the centre
    - Hip lines run from ridge endpoints to the four eave corners
    - All four slopes have the same pitch

    Vertices:
      R1, R2 = ridge endpoints (at ridge_height)
      E1..E4 = eave corners (at wall_height, with overhang offset)

    Faces:
      Long faces:  [E1, E2, R2, R1] and [E3, E4, R1, R2]
      End faces:   [E2, E3, R2]     and [E4, E1, R1]   (triangles)
    """
    import numpy as np

    # Derive building extents from wall perimeter
    xs = [p[0] for p in wall_perimeter_m]
    ys = [p[1] for p in wall_perimeter_m]
    xmin, xmax = min(xs), max(xs)
    ymin, ymax = min(ys), max(ys)
    W = xmax - xmin   # short dimension
    L = ymax - ymin   # long dimension (or swap if L < W)

    # Overhang-extended eave corners
    ex_min = xmin - overhang_m
    ex_max = xmax + overhang_m
    ey_min = ymin - overhang_m
    ey_max = ymax + overhang_m

    # Ridge endpoints (inset from gable eave edges by half-span)
    half_W = W / 2
    rx_min = ex_min + (half_W + overhang_m)
    rx_max = ex_max - (half_W + overhang_m)

    # Build mesh
    mesh = bpy.data.meshes.new("Roof")
    obj  = bpy.data.objects.new("Roof", mesh)
    bpy.context.collection.objects.link(obj)

    bm = bmesh.new()
    z_eave  = wall_height
    z_ridge = ridge_height

    # Eave corners
    E = [
        bm.verts.new((ex_min, ey_max, z_eave)),   # front-left
        bm.verts.new((ex_max, ey_max, z_eave)),   # front-right
        bm.verts.new((ex_max, ey_min, z_eave)),   # back-right
        bm.verts.new((ex_min, ey_min, z_eave)),   # back-left
    ]
    # Ridge
    R = [
        bm.verts.new((rx_min, (ey_min+ey_max)/2, z_ridge)),
        bm.verts.new((rx_max, (ey_min+ey_max)/2, z_ridge)),
    ]
    bm.verts.ensure_lookup_table()

    # Faces
    bm.faces.new([E[0], E[1], R[1], R[0]])   # front slope
    bm.faces.new([E[2], E[3], R[0], R[1]])   # back slope
    bm.faces.new([E[1], E[2], R[1]])          # right hip (triangle)
    bm.faces.new([E[3], E[0], R[0]])          # left hip (triangle)
    # Ridge cap (only if ridge has length)
    if abs(rx_max - rx_min) > 0.01:
        bm.faces.new([R[0], R[1]])            # degenerate — skip; ridge is an edge

    bmesh.ops.recalc_face_normals(bm, faces=bm.faces[:])
    bm.to_mesh(mesh)
    bm.free()
    mesh.update()
```

---

## STEP 8 — Verify in Blender

After building, take viewport screenshots from at least three angles:

```python
import mathutils

# 1. Top-down orthographic (compare against roof plan)
# 2. Perspective from front-exterior (check slope and overhang)
# 3. Perspective from corner (check hip/ridge geometry)

# Quick dimension check
roof_obj = bpy.data.objects['Roof']
print(f"Roof dimensions: {roof_obj.dimensions}")
# Should match building footprint + 2× overhang on each axis
```

**Checklist after build:**
- [ ] Roof sits flush on top of outer walls (no gap, no penetration)
- [ ] Overhang extends equally on all eave sides
- [ ] Ridge is centred over the building (for symmetric roofs)
- [ ] All faces have outward normals (no dark/inside-out faces)
- [ ] No faces are missing (no holes)
- [ ] Ridge height matches the calculated value from Step 4

---

## STEP 9 — Output: Handoff Data Block

```python
print("roof:")
print(f"  type: {roof_type}")
print(f"  wall_height: {WALL_HEIGHT}")
print(f"  ridge_height: {RIDGE_HEIGHT}")
print(f"  overhang_m: {OVERHANG_M}")
print(f"  slope_deg: {slope_angle_deg}")
if ridge:
    print(f"  ridge_px: [{ridge['x1']}, {ridge['y']}] → [{ridge['x2']}, {ridge['y']}]")
    x1m, y1m = px_to_blender(ridge['x1'], ridge['y'])
    x2m, y2m = px_to_blender(ridge['x2'], ridge['y'])
    print(f"  ridge_m:  [{x1m}, {y1m}] → [{x2m}, {y2m}]")
if hip_lines:
    print("  hip_lines:")
    for i, h in enumerate(hip_lines):
        x1m, y1m = px_to_blender(h['x1'], h['y1'])
        x2m, y2m = px_to_blender(h['x2'], h['y2'])
        print(f"    - [{x1m}, {y1m}] → [{x2m}, {y2m}]")
```

---

## COMMON ERRORS AND FIXES

| Error | Symptom | Cause | Fix |
|---|---|---|---|
| Roof too small | Roof sits inside the outer walls | Used wall perimeter without adding overhang | Add `overhang_m` offset to all eave edges outward |
| Roof too flat | Ridge height visually too low | Used wrong slope angle or formula | Re-check slope notation; recalculate `ridge_height_from_slope` |
| Gap at ridge | Two roof faces don't meet at apex | Ridge Y-coordinate not centred between eave edges | Recompute ridge position as midpoint between front and back eave |
| Hip lines don't align | Hip line angles look wrong | Building not rectangular or overhang not uniform | Measure hip endpoints directly from roof plan pixels; don't derive analytically |
| Roof penetrates walls | Roof mesh sits inside the wall body | `wall_height` set to 0 or wrong | Set `wall_height` to the same value used in Phase 3 (typically 2.70m) |
| Wrong scale | Roof plan was at different scale | Assumed same scale as floor plan | Cross-check a known dimension against Phase 1; derive separate `ROOF_SCALE` if different |
| Normals inverted | Roof faces appear dark from above | Face winding reversed | Run `bmesh.ops.recalc_face_normals` after build |
| Dormer modelled incorrectly | Dormer shape doesn't match plan | Only used roof plan, ignored floor plan | Cross-reference dormer position and width with floor plan; model separately |

---

## ARCHITECTURAL DEFAULTS (European)

| Element | Default Value |
|---|---|
| Wall top plate height | 2.70 m (from Phase 3) |
| Eave/overhang depth | 0.40–0.60 m |
| Gable overhang depth | 0.20–0.40 m |
| Common residential slope | 30°–40° |
| Flat roof drainage slope | 1.5°–3° (1–5%) |
| Ridge tile width | 0.20–0.30 m (cosmetic only) |
| Gutter width (cosmetic) | 0.15–0.25 m |

---

## WORKFLOW SUMMARY

```
0. Confirm Phase 3 Blender scene is available (OuterWalls object exists)
   Load Phase 1 handoff → SCALE, OX/OY, wall perimeter metres
1. Load roof plan image — identify roof type, annotate key features visually
2. Extract overhang outline (outermost roof edge)
   Measure overhang offset from wall perimeter
3. Extract ridge line(s) and hip/valley lines
   For hip lines: prefer visual identification over automated detection
4. Read slope annotation → compute ridge height
   If no slope labeled → ask user before proceeding
5. Generate verification image (cyan=overhang, red=ridge, orange=hips)
   Get user approval before building
6. Convert all pixel coordinates → Blender metres using Phase 1 SCALE
7. Build roof mesh in Blender:
     Flat   → single extruded face with slight slope
     Gable  → two rectangular faces + two triangular gable faces
     Hip    → two trapezoidal + two triangular faces
     Complex → decompose into sub-roofs, build each separately
8. Verify in Blender: top-down + two perspective views
   Check flush seating on walls, overhang, normals, ridge height
9. Emit roof YAML handoff block
10. STOP
```
