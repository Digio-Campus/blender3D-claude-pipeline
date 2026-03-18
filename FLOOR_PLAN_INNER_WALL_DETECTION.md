# SKILL: Inner Wall Detection from Architectural Floor Plans
> Applicable to any 2D top-down thin-line architectural floor plan image.
> Designed to be followed by an AI agent using PIL/numpy for pixel analysis.
> **Phase 3 of the pipeline. Requires handoffs from Phase 1 (wall extraction) and Phase 2 (openings).**

---

## OVERVIEW

This is **Phase 3** of the floor plan pipeline. It receives the structured outputs
of Phase 1 (outer wall geometry) and Phase 2 (openings) and uses both to constrain
the search for interior partition walls and interior doors.

Do NOT run this phase without both handoffs. The Phase 1 handoff defines the
interior search zone; the Phase 2 handoff is used to avoid misidentifying known
outer openings as interior wall segments.

Inner walls partition the building's interior into rooms. They differ from outer
walls in three key ways: they are **thinner** (typically a single drawn line or
two closely spaced lines, vs. the wide double-line band of outer walls), they can
**terminate anywhere** in the interior (not just at the perimeter), and they
frequently have **interior doors** drawn as a gap + arc, just like outer doors but
narrower and entirely within the interior zone.

---

## PHASE INPUT — Handoffs from Phase 1 and Phase 2

Load both handoff files before doing anything else.

**Phase 1 handoff (wall extraction):** provides `scale`, `origin`, and the outer
wall segment geometry used to build the interior search zone.

**Phase 2 handoff (openings):** provides the outer openings list, used in Step 3
to prevent re-detecting known outer door gaps as interior wall endpoints.

```python
import yaml
import numpy as np
from PIL import Image

# Phase 1 handoff
with open('wall_extraction_handoff.yaml') as f:
    p1 = yaml.safe_load(f)

SCALE        = p1['scale']           # m/px — used in Step 6
OX, OY       = p1['origin']          # origin pixel — used in Step 6
WALL_SEGMENTS = p1['walls']          # outer wall geometry — used in Step 1

# Phase 2 handoff
with open('openings_handoff.yaml') as f:
    p2 = yaml.safe_load(f)

OUTER_OPENINGS = p2['openings']      # list of outer windows + doors — used in Step 3

img  = Image.open('plan.png').convert('RGB')
arr  = np.array(img)
gray = arr[:, :, 0]
```

**Error propagation notice:** the interior search zone is derived directly from
Phase 1 inner face positions. An error in a Phase 1 wall boundary will shift or
narrow the search zone on that side, potentially missing inner walls that terminate
near it. If inner walls on one side are systematically missed, re-check the
corresponding Phase 1 inner face before debugging this phase.

---

## CRITICAL WARNINGS — READ BEFORE STARTING

### 1. Inner walls are thin — do not expect a double-line band
Outer walls have a wide, clearly visible band (two parallel lines separated by
10–20px of wall thickness). Inner walls are typically drawn as a **single dark
line** 1–3px wide, or two lines separated by only 3–8px. The detection approach
here is projection-based, not band-based. Do not apply outer wall logic here.

### 2. Furniture is the main source of false positives
Sofas, beds, tables, and kitchen units are drawn with straight dark lines that
can look exactly like short inner wall segments. The primary filter is **length**:
architectural inner walls almost always span at least 40px (≈ 0.6 m). Reject any
candidate shorter than 40px without visual confirmation.

### 3. Stairs, hatching, and annotation lines create noise
Staircase hatching produces dense, evenly-spaced parallel lines — they will spike
the projection. Dimension lines and section marks also appear as dark lines.
Visually confirm every candidate before accepting. When in doubt, zoom.

### 4. Interior doors look identical to outer doors
Interior doors are a gap in the inner wall line plus a quarter-circle arc, drawn
exactly like outer doors. They are detected in the same way (gap edges in the
wall line) but are typically narrower (0.70–0.90 m vs 0.80–1.00 m outer). Always
check for an arc in the zoomed crop before classifying a gap as an interior door.

### 5. T-junctions and L-junctions create projection bumps
Where two inner walls meet, or where an inner wall meets an outer wall's inner
face, there is a junction. These junctions create short spikes in the projection
that can look like separate short segments. Merge segments separated by fewer
than 8px before applying the length filter.

### 6. The interior zone margin matters
Scan strictly from the inner face of each outer wall (not the outer face, not the
midpoint). The inner face positions come from the Phase 1 handoff. Add a 2px
inward margin to avoid picking up the inner face line itself as a false candidate.

### 7. Inner wall endpoints must sit at the outer wall INNER FACE — not inside the wall body

⚠️ **This is the single most common 3D modelling error from this phase.**

When an inner wall terminates at an outer wall, its endpoint must be placed
**exactly at the inner face** of the outer wall — not at the wall centre, not past it.

**Why it matters:** Outer walls have a cavity of thickness T (e.g. 0.30m). Window
openings cut completely through this cavity from outer face to inner face. If an
inner wall endpoint is positioned anywhere between the outer face and the inner
face, the end cap of that inner wall sits **inside the window hole** and is fully
visible from outside through the window opening.

```
WRONG — endpoint at wall centre (y = 0.000):
  ────────┬─────────────┬────────────
  outer   │  end cap    │  inner
  face    │  VISIBLE!   │  face
  ────────┴─────────────┴────────────
              ↑ window cuts here

CORRECT — endpoint at inner face (y = -0.150):
  ──────────────────────┬────────────
  outer                 │  end cap
  face                  │  flush ✓
  ──────────────────────┴────────────
                          ↑ inner face
```

**The rule:** For every inner wall endpoint that touches an outer wall, set its
coordinate to exactly the **inner face** coordinate of that outer wall:

```python
# Outer wall thickness T = 0.30m, half = H = 0.15m
# Top wall:    outer face y=+0.150,  inner face y=-0.150  → endpoint y = -0.150
# Bottom wall: outer face y=-10.500, inner face y=-10.200 → endpoint y = -10.200
# Left wall:   outer face x=-0.150,  inner face x=+0.150  → endpoint x = +0.150
# Step walls:  inner face = wall_centre ± H (inward direction)

# General formula:
inner_face = wall_centre - H  # for walls where interior is in -Y direction
inner_face = wall_centre + H  # for walls where interior is in +X direction
```

**Do NOT** push the endpoint to the wall centre or past it just to "ensure contact".
That places the end cap inside the window cavity. At the inner face, the end cap
is flush with the interior surface — cleanly embedded in the solid wall body on
either side of any window.

---

## STEP 1 — Build the Interior Search Zone

From the Phase 1 handoff, derive the bounding rectangle of the interior — the
region strictly inside all outer wall inner faces. This is the only zone scanned
in Steps 2 and 3.

For L-shaped or stepped building footprints, the interior zone is not a single
rectangle. Derive one bounding box per convex sub-region if needed, but for
typical rectangular plans a single box suffices.

```python
def build_interior_zone(wall_segments):
    """
    Derive the interior search bounding box from the inner faces of all
    outer wall segments.

    Returns (col_min, row_min, col_max, row_max) — the pixel rectangle
    strictly inside all outer wall inner faces, with a 2px inward margin.
    """
    top_inner    = None   # largest row among top-wall inner faces (most downward)
    bottom_inner = None   # smallest row among bottom-wall inner faces (most upward)
    left_inner   = None   # largest col among left-wall inner faces (most rightward)
    right_inner  = None   # smallest col among right-wall inner faces (most leftward)

    for seg in wall_segments:
        if seg['orientation'] == 'horizontal':
            inner = seg['inner_face_row']
            if seg['outer_face_row'] < inner:   # top wall: outer above inner
                top_inner = max(top_inner or 0, inner)
            else:                                # bottom wall: outer below inner
                bottom_inner = min(bottom_inner or 9999, inner)
        else:  # vertical
            inner = seg['inner_face_col']
            if seg['outer_face_col'] < inner:   # left wall: outer left of inner
                left_inner = max(left_inner or 0, inner)
            else:                               # right wall: outer right of inner
                right_inner = min(right_inner or 9999, inner)

    MARGIN = 2
    return (
        left_inner   + MARGIN,
        top_inner    + MARGIN,
        right_inner  - MARGIN,
        bottom_inner - MARGIN,
    )

INTERIOR = build_interior_zone(WALL_SEGMENTS)
col_min, row_min, col_max, row_max = INTERIOR

print(f"Interior search zone: cols [{col_min}–{col_max}], rows [{row_min}–{row_max}]")
```

For stepped or L-shaped plans, call `build_interior_zone` once per convex
sub-region and union the results. Scan each sub-region independently in Step 2.

---

## STEP 2 — Projection Scan: Candidate Line Detection

Project dark pixel counts along each row (to find horizontal inner walls) and
each column (to find vertical inner walls). A row or column with a high dark
pixel count running across a significant portion of the interior indicates an
inner wall at that position.

```python
def projection_scan(gray, col_min, row_min, col_max, row_max,
                    thresh=140, min_coverage=0.35):
    """
    Scan the interior zone for rows and columns that are predominantly dark.

    min_coverage: fraction of the span that must be dark for a row/col
    to be considered a candidate wall position. 0.35 = 35% dark pixels.
    Lower this if walls are expected to be short; raise it to suppress noise.

    Returns:
      h_candidates: list of row indices with high horizontal dark coverage
      v_candidates: list of col indices with high vertical dark coverage
    """
    h_span = col_max - col_min
    v_span = row_max - row_min

    h_candidates = []
    for row in range(row_min, row_max):
        strip = gray[row, col_min:col_max]
        dark_count = np.sum(strip < thresh)
        if dark_count / h_span >= min_coverage:
            h_candidates.append(row)

    v_candidates = []
    for col in range(col_min, col_max):
        strip = gray[row_min:row_max, col]
        dark_count = np.sum(strip < thresh)
        if dark_count / v_span >= min_coverage:
            v_candidates.append(col)

    return h_candidates, v_candidates

h_candidates, v_candidates = projection_scan(gray, col_min, row_min, col_max, row_max)
print(f"Horizontal candidate rows: {h_candidates}")
print(f"Vertical candidate cols:   {v_candidates}")
```

**Tuning `min_coverage`:**
- Start at `0.35`. If too many furniture lines appear, raise to `0.50`.
- If known walls are missed (wall doesn't span the full interior width), lower
  to `0.20` and rely more heavily on the length filter in Step 3.

---

## STEP 3 — Segment Tracing: Find Wall Extents

For each candidate row/column, trace along it to find where the dark line
actually starts and ends. Group consecutive candidate rows/columns into single
wall positions, then find the continuous dark run within each.

```python
def cluster_positions(positions, gap=4):
    """
    Group consecutive positions (rows or cols) that are within `gap` of each
    other into clusters. Returns list of (center, members) tuples.
    """
    if not positions:
        return []
    clusters = [[positions[0]]]
    for p in positions[1:]:
        if p - clusters[-1][-1] <= gap:
            clusters[-1].append(p)
        else:
            clusters.append([p])
    return [(int(np.median(c)), c) for c in clusters]

def trace_segment(gray, fixed_pos, scan_start, scan_end,
                  orientation, thresh=140, min_length=40, gap_tolerance=8):
    """
    Trace a dark line along a row (orientation='horizontal') or column
    (orientation='vertical') to find its pixel extents.

    fixed_pos:   the row (horizontal) or col (vertical) to trace along
    scan_start:  start of the scan range (col_min or row_min)
    scan_end:    end of the scan range (col_max or row_max)
    gap_tolerance: max gap of light pixels allowed within a single segment
    min_length:  discard segments shorter than this (px)

    Returns a list of (start, end) tuples for each valid dark run.
    """
    if orientation == 'horizontal':
        line = gray[fixed_pos, scan_start:scan_end]
    else:
        line = gray[scan_start:scan_end, fixed_pos]

    segments = []
    in_seg = False
    seg_start = 0
    gap_run = 0

    for i, val in enumerate(line):
        if val < thresh:
            if not in_seg:
                seg_start = i
                in_seg = True
            gap_run = 0
        else:
            if in_seg:
                gap_run += 1
                if gap_run > gap_tolerance:
                    seg_end = i - gap_run
                    length = seg_end - seg_start
                    if length >= min_length:
                        segments.append((scan_start + seg_start,
                                         scan_start + seg_end))
                    in_seg = False
                    gap_run = 0

    if in_seg:
        seg_end = len(line) - gap_run
        if (seg_end - seg_start) >= min_length:
            segments.append((scan_start + seg_start, scan_start + seg_end))

    return segments


# Collect raw inner wall candidates
raw_walls = []

h_clusters = cluster_positions(h_candidates)
for (row, _) in h_clusters:
    segs = trace_segment(gray, row, col_min, col_max, 'horizontal')
    for (c_start, c_end) in segs:
        raw_walls.append({
            'orientation': 'horizontal',
            'fixed':       row,         # the row the wall sits on
            'start':       c_start,     # col where the wall starts
            'end':         c_end,       # col where the wall ends
        })

v_clusters = cluster_positions(v_candidates)
for (col, _) in v_clusters:
    segs = trace_segment(gray, col, row_min, row_max, 'vertical')
    for (r_start, r_end) in segs:
        raw_walls.append({
            'orientation': 'vertical',
            'fixed':       col,         # the col the wall sits on
            'start':       r_start,     # row where the wall starts
            'end':         r_end,       # row where the wall ends
        })

print(f"Raw inner wall candidates: {len(raw_walls)}")
for w in raw_walls:
    span = w['end'] - w['start']
    print(f"  {w['orientation']:10s}  fixed={w['fixed']}  "
          f"[{w['start']} → {w['end']}]  span={span}px")
```

**After tracing, filter out candidates that overlap known outer openings:**

```python
def overlaps_outer_opening(wall, outer_openings, margin=5):
    """
    Return True if a raw inner wall candidate coincides with a known outer
    opening (door gap already detected in Phase 2). This prevents a door
    gap in an outer wall from being re-detected as an inner wall stub.
    """
    for op in outer_openings:
        # Compare only same-orientation openings
        op_orient = 'horizontal' if op['wall'] in ('top', 'bottom') else 'vertical'
        if op_orient != wall['orientation']:
            continue
        op_start = op['px'][0][0] if wall['orientation'] == 'horizontal' \
                   else op['px'][0][1]
        op_end   = op['px'][1][0] if wall['orientation'] == 'horizontal' \
                   else op['px'][1][1]
        # Check for overlap
        if wall['start'] - margin < op_end and wall['end'] + margin > op_start:
            return True
    return False

filtered_walls = [w for w in raw_walls
                  if not overlaps_outer_opening(w, OUTER_OPENINGS)]
print(f"After outer-opening filter: {len(filtered_walls)} candidates")
```

---

## STEP 4 — Visual Confirmation with Zoomed Crops

Save a zoomed crop for every candidate before accepting it. Inner wall crops
need more context than opening crops — include at least 30px padding on each
side so the termination points and any junction geometry are visible.

```python
def save_inner_wall_crop(img, wall, label, pad=30):
    """Save a zoomed crop centred on one inner wall candidate."""
    if wall['orientation'] == 'horizontal':
        x1 = wall['start'] - pad
        x2 = wall['end']   + pad
        y1 = wall['fixed'] - 20
        y2 = wall['fixed'] + 20
        scale_w, scale_h = 4, 8
    else:
        x1 = wall['fixed'] - 20
        x2 = wall['fixed'] + 20
        y1 = wall['start'] - pad
        y2 = wall['end']   + pad
        scale_w, scale_h = 8, 4

    crop = img.crop((x1, y1, x2, y2))
    crop = crop.resize((crop.width * scale_w, crop.height * scale_h), Image.NEAREST)
    crop.save(f'zoom_iwall_{label}.png')

for i, wall in enumerate(filtered_walls):
    save_inner_wall_crop(img, wall, str(i))
```

**What to look for in the crop:**

- ✅ A continuous dark line running cleanly across the interior → inner wall
- ✅ Line terminates at an outer wall inner face → correctly bounded
- ✅ Line terminates in a T- or L-junction with another inner wall → valid
- ⚠️  Line is shorter than expected — check both endpoints; one may be clipped
  by a junction. If so, re-run `trace_segment` with a larger `gap_tolerance`.
- ❌ Multiple short parallel lines in a small area → stair hatching or furniture.
  Discard the whole cluster.
- ❌ Line is very short (< 60px) with no connection to any wall or other segment →
  likely furniture edge or dimension line. Discard.
- ❌ Line is irregular or diagonal → annotation or hatch line. Discard.

**Interior door detection during visual confirmation:**

While inspecting each crop, check whether the wall line has a **gap** anywhere
along its length. A gap in an inner wall line + a quarter-circle arc = interior
door. Record it immediately:

```python
# During visual review, if a gap + arc is spotted:
interior_door_candidates.append({
    'wall_index': i,          # which inner wall it belongs to
    'gap_approx': (col, col), # estimated gap bounds from the crop
    'needs_precise_edges': True,
})
```

Precise gap edges are measured in Step 5.

---

## STEP 5 — Interior Door Gap Edge Detection

For any inner wall confirmed to have an interior door, find the exact gap edges
using the same approach as Phase 2's `find_door_gap_edges`, but scanning along
the inner wall's `fixed` row or column.

```python
def find_inner_door_gap(gray, wall, gap_approx_start, gap_approx_end,
                        search_pad=15, thresh=160):
    """
    Find the exact pixel positions where an inner wall line vanishes (door gap).
    Works identically to Phase 2 find_door_gap_edges but operates on interior lines.

    wall:              a confirmed inner wall dict (orientation, fixed, start, end)
    gap_approx_start:  rough start of the gap (from visual inspection)
    gap_approx_end:    rough end of the gap

    Returns (gap_start, gap_end) or None.
    """
    fixed = wall['fixed']

    if wall['orientation'] == 'horizontal':
        scan_range = range(gap_approx_start - search_pad,
                           gap_approx_end   + search_pad)
        is_dark = lambda pos: gray[fixed, pos] < thresh
    else:
        scan_range = range(gap_approx_start - search_pad,
                           gap_approx_end   + search_pad)
        is_dark = lambda pos: gray[pos, fixed] < thresh

    gap_start, gap_end = None, None
    in_gap = False
    for pos in scan_range:
        dark = is_dark(pos)
        if not dark and not in_gap:
            gap_start = pos
            in_gap = True
        elif dark and in_gap:
            gap_end = pos - 1
            break

    if gap_start is None or gap_end is None:
        return None
    return gap_start, gap_end


confirmed_interior_doors = []
for dc in interior_door_candidates:
    wall = confirmed_walls[dc['wall_index']]
    result = find_inner_door_gap(
        gray, wall, dc['gap_approx'][0], dc['gap_approx'][1])
    if result is None:
        print(f"  WARNING: could not confirm interior door gap on wall {dc['wall_index']}"
              f" — inspect zoom crop manually")
        continue
    gap_start, gap_end = result
    confirmed_interior_doors.append({
        'name':        f"ID{len(confirmed_interior_doors)+1}",
        'wall':        wall['name'],
        'orientation': wall['orientation'],
        'fixed':       wall['fixed'],
        'gap_start':   gap_start,
        'gap_end':     gap_end,
    })
    print(f"  Interior door on {wall['name']}: gap [{gap_start} → {gap_end}]"
          f"  ({round((gap_end - gap_start) * SCALE, 3)} m)")
```

---

## STEP 6 — Verification Image

Draw all confirmed inner walls and interior doors over the original plan in a
**single image**, using colours that do not conflict with the Phase 2 verification
image (orange = windows, blue = outer doors).

- **Green** `(34, 139, 34)` → inner walls
- **Violet** `(148, 0, 211)` → interior doors

```python
from PIL import Image, ImageDraw

img_verify = Image.open('plan.png').convert('RGB')
draw = ImageDraw.Draw(img_verify)

GREEN  = (34, 139, 34)
VIOLET = (148, 0, 211)

def mark_inner_wall(wall, label):
    if wall['orientation'] == 'horizontal':
        x1, y1 = wall['start'], wall['fixed'] - 3
        x2, y2 = wall['end'],   wall['fixed'] + 3
    else:
        x1, y1 = wall['fixed'] - 3, wall['start']
        x2, y2 = wall['fixed'] + 3, wall['end']
    draw.rectangle([x1, y1, x2, y2], outline=GREEN, width=2)
    if label:
        draw.rectangle([x1, y1-18, x1+len(label)*9+4, y1-2], fill=(255,255,255))
        draw.text((x1+3, y1-17), label, fill=GREEN)

def mark_interior_door(door):
    fixed = door['fixed']
    if door['orientation'] == 'horizontal':
        x1, y1 = door['gap_start'], fixed - 5
        x2, y2 = door['gap_end'],   fixed + 5
    else:
        x1, y1 = fixed - 5, door['gap_start']
        x2, y2 = fixed + 5, door['gap_end']
    draw.rectangle([x1, y1, x2, y2], outline=VIOLET, width=2)
    draw.rectangle([x1, y1-18, x1+len(door['name'])*9+4, y1-2], fill=(255,255,255))
    draw.text((x1+3, y1-17), door['name'], fill=VIOLET)

for wall in confirmed_walls:
    mark_inner_wall(wall, wall['name'])

for door in confirmed_interior_doors:
    mark_interior_door(door)

img_verify.save('inner_walls_verification.png')
```

**Do NOT proceed to Step 7 until the user has approved the verification image.**
Every green segment and every violet door must be reviewed. Missed walls or
false positives discovered here must be corrected before coordinate conversion.

---

## STEP 6b — Endpoint Validation Against Outer Walls and Windows

Before converting to metres, run a numerical check on every endpoint that
touches an outer wall. Two checks must pass:

### Check 1 — No gaps (endpoint reaches the outer wall)

The endpoint coordinate must be at or past the outer wall inner face (i.e. inside
the wall body). A coordinate that stops short leaves a physical gap between the
inner wall and the outer wall.

```python
def check_no_gap(endpoint_val, inner_face_val, direction):
    """
    direction: 'neg' if interior is in the negative axis direction (e.g. top wall)
               'pos' if interior is in the positive axis direction (e.g. left wall)
    Returns gap distance (0.0 = no gap, >0 = gap in metres).
    """
    if direction == 'neg':
        gap = inner_face_val - endpoint_val   # positive = endpoint short of face
    else:
        gap = endpoint_val - inner_face_val   # positive = endpoint short of face
    return max(0.0, gap)
```

### Check 2 — Not inside the window cavity (endpoint not past the inner face)

The endpoint must NOT be positioned between the outer face and inner face of the
outer wall. That zone is the window cavity — any end cap there is visible from
outside through every window on that wall.

```python
def check_not_in_cavity(endpoint_val, outer_face_val, inner_face_val):
    """
    Returns True if endpoint is inside the window cavity (bad), False if OK.
    """
    lo = min(outer_face_val, inner_face_val)
    hi = max(outer_face_val, inner_face_val)
    return lo < endpoint_val < hi   # strictly inside = problem
```

### Combined validation loop

```python
# Define outer wall geometry: (name, centre, inner_face, outer_face, axis, direction)
outer_walls = {
    'top':    dict(centre=  0.000, inner= -0.150, outer= +0.150, axis='y', dir='neg'),
    'bottom': dict(centre=-10.350, inner=-10.200, outer=-10.500, axis='y', dir='pos'),
    'left':   dict(centre=  0.000, inner= +0.150, outer= -0.150, axis='x', dir='pos'),
    # add step walls as needed
}

for wall in confirmed_walls:
    for endpoint_label, endpoint_val, wall_name in get_outer_endpoints(wall):
        w = outer_walls[wall_name]
        gap = check_no_gap(endpoint_val, w['inner'], w['dir'])
        cavity = check_not_in_cavity(endpoint_val, w['outer'], w['inner'])

        if gap > 0.001:
            print(f"⚠ GAP: {wall['name']} {endpoint_label}: {gap:.3f}m short of {wall_name} inner face")
            print(f"  Fix: set endpoint to {w['inner']:.3f}")
        elif cavity:
            print(f"⚠ CAVITY: {wall['name']} {endpoint_label}: endpoint {endpoint_val:.3f} is inside {wall_name} body!")
            print(f"  Fix: set endpoint to {w['inner']:.3f}")
        else:
            print(f"  ✓ {wall['name']} {endpoint_label}: OK")
```

**Both checks must pass for every endpoint before proceeding to Step 7.**
The correct value for any endpoint that touches an outer wall is always exactly
the outer wall's inner face coordinate — no more, no less.

---

## STEP 7 — Output: Handoff Data Block

`SCALE`, `OX`, and `OY` were loaded from the Phase 1 handoff — do not re-derive
them. Emit two lists: `inner_walls` and `interior_doors`. The 3D modelling phase
uses these to extrude partition walls and cut interior door openings.

```python
# SCALE, OX, OY already loaded from Phase 1 handoff in PHASE INPUT section

def px_to_m(col, row):
    """Convert pixel coords to plan-space metres (Y positive downward)."""
    x = round((col - OX) * SCALE, 3)
    y = round((row - OY) * SCALE, 3)
    return x, y

print("inner_walls:")
for wall in confirmed_walls:
    if wall['orientation'] == 'horizontal':
        c1, r1 = wall['start'], wall['fixed']
        c2, r2 = wall['end'],   wall['fixed']
    else:
        c1, r1 = wall['fixed'], wall['start']
        c2, r2 = wall['fixed'], wall['end']
    x1, y1 = px_to_m(c1, r1)
    x2, y2 = px_to_m(c2, r2)
    print(f"  - name: {wall['name']}")
    print(f"    orientation: {wall['orientation']}")
    print(f"    px:   [{c1}, {r1}] → [{c2}, {r2}]")
    print(f"    m:    x=[{x1}, {x2}]  y=[{y1}, {y2}]")

print("\ninterior_doors:")
for door in confirmed_interior_doors:
    if door['orientation'] == 'horizontal':
        c1, r1 = door['gap_start'], door['fixed']
        c2, r2 = door['gap_end'],   door['fixed']
    else:
        c1, r1 = door['fixed'], door['gap_start']
        c2, r2 = door['fixed'], door['gap_end']
    x1, y1 = px_to_m(c1, r1)
    x2, y2 = px_to_m(c2, r2)
    print(f"  - name: {door['name']}")
    print(f"    wall: {door['wall']}")
    print(f"    px:   [{c1}, {r1}] → [{c2}, {r2}]")
    print(f"    m:    x=[{x1}, {x2}]  y=[{y1}, {y2}]")
```

**Example output format:**

```yaml
inner_walls:
  - name: IW1
    orientation: horizontal
    px:   [201, 320] → [870, 320]
    m:    x=[0.240, 10.275]  y=[3.930, 3.930]

  - name: IW2
    orientation: vertical
    px:   [540, 320] → [540, 740]
    m:    x=[5.325, 5.325]  y=[3.930, 10.230]

  - name: IW3
    orientation: horizontal
    px:   [540, 510] → [870, 510]
    m:    x=[5.325, 10.275]  y=[6.780, 6.780]

interior_doors:
  - name: ID1
    wall: IW1
    px:   [380, 320] → [443, 320]
    m:    x=[2.685, 3.630]  y=[3.930, 3.930]
```

**Do NOT proceed to Blender.** Hand this data block together with the Phase 1
and Phase 2 handoffs to the 3D modelling phase
(BLENDER_FLOORPLAN_3D_MODELING.md).

---

## COMMON ERRORS AND FIXES

| Error | Symptom | Cause | Fix |
|---|---|---|---|
| Furniture detected as wall | Short horizontal segment near a room edge | Furniture line meets coverage threshold | Discard candidates < 60px unless clearly connected to another wall; visually confirm |
| Stair hatch produces many candidates | Dense cluster of parallel horizontal candidates at ~equal spacing | Staircase hatching | In the crop, if lines are evenly spaced and there are 4+ of them → discard the whole cluster |
| Wall clipped at junction | Segment ends 5–10px short of outer wall inner face | gap_tolerance too small for the junction pixel density | Increase gap_tolerance to 12 in trace_segment for that wall |
| Wall not detected | Known wall absent from candidates | Wall doesn't reach min_coverage threshold (e.g., wall spans only half the interior) | Lower min_coverage to 0.20; run trace_segment manually at the expected row/col |
| Outer door gap re-detected as inner wall | A short inner wall stub appears at an outer door location | Door gap in outer wall looks like a floating line from inside | overlaps_outer_opening filter should catch this; check that Phase 2 px coords parse correctly |
| Interior door gap not found | find_inner_door_gap returns None | gap_approx from visual inspection was off | In the crop, re-estimate the gap position more carefully; adjust gap_approx and re-run |
| Interior door too wide | Door width > 1.00 m | gap_approx set too wide from a rough visual estimate | Re-inspect the crop at full zoom; narrow gap_approx to the arc attachment points |
| Wrong interior zone bounds | Candidates appear outside expected area | Phase 1 inner face misidentified for one wall | Re-check the Phase 1 handoff inner_face_row/col for that wall segment |
| Phase 1 error propagated | One full side of the interior is unscanned | build_interior_zone produces a degenerate box | Print INTERIOR and verify all four bounds make geometric sense before scanning |
| Inner wall end cap visible from outside through window | In 3D viewport, looking through a window opening reveals the flat end face of an inner wall | Endpoint was placed at the outer wall centre or past it — inside the window cavity | Move endpoint to exactly the outer wall INNER FACE coordinate. The cavity spans [outer_face, inner_face]; any endpoint strictly between them is wrong. See Warning 7 and Step 6b. |
| Gap between inner wall and outer wall | In 3D viewport, a thin crack of light visible between inner wall end and outer wall surface | Endpoint placed short of the outer wall inner face | Move endpoint to exactly the outer wall inner face. Ensure `check_no_gap` returns 0. |
| Endpoint pushed too far (into cavity) when fixing a gap | Tried to fix a gap by pushing endpoint to wall centre — now end cap is visible through windows | Confused "inside the wall body" with "past the inner face" | The only correct target is the inner face. Centre is too far. See Warning 7. |

---

## ARCHITECTURAL DEFAULTS (European)

| Element | Default Value |
|---|---|
| Inner wall (partition) height | 2.50 m (full ceiling height) |
| Inner wall thickness (typical) | 0.10–0.15 m |
| Interior door width | 0.70–0.90 m |
| Interior door height | 2.10 m |

---

## WORKFLOW SUMMARY

```
0. Load Phase 1 handoff → SCALE, OX/OY, WALL_SEGMENTS
   Load Phase 2 handoff → OUTER_OPENINGS
1. build_interior_zone from WALL_SEGMENTS inner faces → INTERIOR bounding box
2. projection_scan across INTERIOR:
     h_candidates → rows with significant horizontal dark coverage
     v_candidates → cols with significant vertical dark coverage
3. cluster_positions + trace_segment on each candidate:
     → raw inner wall segments with (orientation, fixed, start, end)
   Filter with overlaps_outer_opening to remove Phase 2 door gaps
4. For each filtered candidate:
     Save a zoomed crop (30px padding) and inspect visually
     Accept → inner wall; reject → furniture / hatch / annotation
     If a gap + arc is visible within the wall → flag as interior door candidate
5. For each interior door candidate:
     find_inner_door_gap → precise gap edges (gap_start, gap_end)
6. Generate ONE verification image:
     Green rectangles  = confirmed inner walls
     Violet rectangles = confirmed interior doors
     Get user approval before proceeding
6b. Run endpoint validation (Step 6b) on every endpoint that touches an outer wall:
     check_no_gap     → endpoint must reach the outer wall inner face
     check_not_in_cavity → endpoint must NOT be between outer face and inner face
     Correct target for all outer-wall endpoints = exactly the inner face coordinate
     Any endpoint inside the cavity will be visible from outside through windows
7. Convert all pixel bounds → plan-space metres using SCALE and OX/OY
8. Emit inner_walls and interior_doors YAML blocks
9. STOP — hand off all three phase outputs to the 3D modelling phase
```
