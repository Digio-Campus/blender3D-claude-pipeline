# SKILL: Window & Door Extraction from Architectural Floor Plans
> Applicable to any 2D top-down thin-line architectural floor plan image.
> Designed to be followed by an AI agent using PIL/numpy for pixel analysis.
> **Phase 2 of the pipeline. Requires the Phase 1 handoff from FLOOR_PLAN_WALL_EXTRACTION.md.**

---

## OVERVIEW

This is **Phase 2** of the floor plan pipeline. It receives the structured output
of Phase 1 (outer wall extraction) and uses it directly to constrain and guide the
search for windows and doors. Do NOT run this phase without the Phase 1 handoff —
wall band positions, scale, and origin are consumed here, not re-derived.

Windows and outer doors are encoded in the wall band as a specific visual symbol.
This skill teaches you to find them **precisely**, including their exact pixel bounds,
so they can be converted to real-world coordinates and cut into a 3D model.

---

## PHASE INPUT — Handoff from Phase 1 (Wall Extraction)

Before doing anything else, load the YAML handoff block emitted by Phase 1.
It contains everything this phase needs to scope its search correctly.

**Expected handoff structure:**

```yaml
scale: 0.015        # metres per pixel
origin: [185, 58]   # [col, row] of top-left building corner in the image

walls:
  - name: top
    orientation: horizontal   # horizontal | vertical
    outer_face_row: 58        # for horizontal walls: the row of the outer face line
    inner_face_row: 74        # for horizontal walls: the row of the inner face line
    col_start: 185            # pixel extent along the wall's long axis
    col_end: 1210

  - name: bottom
    orientation: horizontal
    outer_face_row: 758
    inner_face_row: 740
    col_start: 185
    col_end: 870

  - name: left
    orientation: vertical
    outer_face_col: 185       # for vertical walls: the col of the outer face line
    inner_face_col: 201       # for vertical walls: the col of the inner face line
    row_start: 58
    row_end: 758

  - name: right_step2_vertical
    orientation: vertical
    outer_face_col: 1205
    inner_face_col: 1174
    row_start: 58
    row_end: 350
  # ... one entry per outer wall segment
```

**Load it in Python before any scanning:**

```python
import yaml
import numpy as np
from PIL import Image

with open('wall_extraction_handoff.yaml') as f:
    handoff = yaml.safe_load(f)

SCALE = handoff['scale']          # m/px — used in Step 6
OX, OY = handoff['origin']        # origin pixel — used in Step 6
WALL_SEGMENTS = handoff['walls']  # list of dicts — drives all scan ranges below

img = Image.open('plan.png').convert('RGB')
arr = np.array(img)
gray = arr[:, :, 0]
```

**Error propagation notice:** the scan zones used in every step below are derived
directly from `WALL_SEGMENTS`. If Phase 1 has a boundary error on a wall segment,
openings on that segment may be mis-located. If results look wrong on one wall,
re-check the corresponding Phase 1 output before debugging this phase.

---

## CRITICAL WARNINGS — READ BEFORE STARTING

### 1. Each architect draws differently
There is no universal convention. Some plans use 3 lines, some use 4, some use a
filled grey band. Do NOT rely solely on line-count heuristics. **Always visually
confirm with zoomed crops** before committing to any window position.

### 2. Line-count heuristics are a starting point, not the answer
Counting dark lines in the wall band (2 = solid, 3–4 = window) is useful for a
first pass, but generates many false positives:
- Interior walls running close and parallel to the outer wall look like a 3rd line
- Corner junctions produce extra lines
- Furniture symbols drawn against the inner wall face can mimic glass panes

**Always cross-check with visual inspection of zoomed crops.**

### 3. The bump is the ground truth
In this drawing style, every window opening is flanked by two small perpendicular
rectangles — **jamb indicators** (also called "bumps") — that cross the wall band
at the exact start and end of the glazing. These are the definitive markers:

```
Wall band cross-section at a window endpoint:

  ────────────────────────────┤ ├────────────────────
  outer face                  │ │  ← bump (jamb)
  ────────────────────────────┤ ├────────────────────
                           glass pane lines here →
  ────────────────────────────┤ ├────────────────────
  inner face                  │ │
  ────────────────────────────┤ ├────────────────────
```

**Never use any other feature to define the window boundary. The bumps are
the only reliable, architect-consistent marker.**

### 4. The bump has TWO faces — use the outer one
Each bump is itself a small rectangle, a few pixels wide. It has an outer face
and an inner face. Always use the **outermost dark pixel** of the bump as the
window boundary — not the inner face, which is a few pixels inward.

### 5. The window symbol can be wider than the wall band
For windows on vertical walls, the glazing lines may extend slightly beyond the
nominal wall band columns on the interior side. Always scan a few pixels wider
than expected in both directions when looking for the left/right extent.

### 6. The door symbol: gap + arc
Doors are drawn as a **gap** in the outer wall face line — the line disappears
entirely across the door width — plus a **quarter-circle arc** that shows the
door swing direction. The inner face line may also vanish or thin out.

```
Door symbol (horizontal wall, door swings inward):

  ────────────────        ────────────────
  outer face      ← GAP →  outer face
  ────────────────        ────────────────
         ╲                           (arc not shown in ASCII)
          ╲ arc
  ─────────────────────────────────────────
  inner face (may be continuous or gapped)
```

The primary detector is the **outer face gap** — not the arc. The arc confirms
the opening is a door and reveals swing direction, but it is drawn inconsistently
across architects (some omit it entirely on entrance doors).

Windows keep both wall face lines intact and add glass pane lines between them.
**If the outer face line vanishes → door. If it stays intact → window.**

### 7. The arc is a confirmation signal, not the primary detector
Do not try to detect the arc first. Arcs vary in radius, thickness, and stroke
style. Some architects draw them as solid curves; others as dashed lines; some
main entrance doors omit them altogether. Always detect the gap first, then look
for the arc as a secondary confirmation in the zoomed crop. If there is a clear
gap but no visible arc, treat it as a door anyway and flag it for user review.

---

## STEP 1 — Build the Wall Band Registry from the Handoff

Do **not** re-measure wall positions from the image. Use `WALL_SEGMENTS` directly.
For each segment, derive the glass pane zone (where window symbols appear) and the
between-face zone (where bump scans will run).

```python
def build_band_registry(wall_segments):
    """
    Convert Phase 1 wall segment dicts into scan-ready band descriptors.
    Returns a list of dicts, one per segment.
    """
    registry = []
    for seg in wall_segments:
        entry = {'name': seg['name'], 'orientation': seg['orientation']}

        if seg['orientation'] == 'horizontal':
            outer = seg['outer_face_row']
            inner = seg['inner_face_row']
            # Normalise so row_top < row_bottom regardless of which face is higher
            row_top  = min(outer, inner)
            row_bot  = max(outer, inner)
            entry.update({
                'outer_face': outer,
                'inner_face': inner,
                'row_top':    row_top,
                'row_bot':    row_bot,
                'glass_row_start': row_top + 1,   # strictly between the face lines
                'glass_row_end':   row_bot - 1,
                'scan_col_start':  seg['col_start'],
                'scan_col_end':    seg['col_end'],
            })

        else:  # vertical
            outer = seg['outer_face_col']
            inner = seg['inner_face_col']
            col_left  = min(outer, inner)
            col_right = max(outer, inner)
            entry.update({
                'outer_face': outer,
                'inner_face': inner,
                'col_left':   col_left,
                'col_right':  col_right,
                'glass_col_start': col_left  + 1,
                'glass_col_end':   col_right - 1,
                # Scan 15px beyond the interior face (see Critical Warning #5)
                'scan_col_start':  col_left  - 15,
                'scan_col_end':    col_right + 15,
                'scan_row_start':  seg['row_start'],
                'scan_row_end':    seg['row_end'],
            })

        registry.append(entry)
    return registry

band_registry = build_band_registry(WALL_SEGMENTS)
```

Every subsequent step iterates over `band_registry`. Scan boundaries come from
here — never hardcode pixel positions.

---

## STEP 2 — First Pass: Line Count Heuristic

Use this to get candidate opening locations quickly. Do NOT trust it blindly.
Iterate over `band_registry` — scan ranges come directly from Step 1.

```python
def count_dark_lines_in_col(gray, col, row_start, row_end, thresh=140):
    """Count distinct dark lines crossing a vertical slice of a horizontal wall."""
    lines = 0
    in_dark = False
    for r in range(row_start, row_end):
        if gray[r, col] < thresh:
            if not in_dark:
                lines += 1
                in_dark = True
        else:
            in_dark = False
    return lines

def count_dark_lines_in_row(gray, row, col_start, col_end, thresh=140):
    """Count distinct dark lines crossing a horizontal slice of a vertical wall."""
    lines = 0
    in_dark = False
    for c in range(col_start, col_end):
        if gray[row, c] < thresh:
            if not in_dark:
                lines += 1
                in_dark = True
        else:
            in_dark = False
    return lines

def find_candidates(gray, band, thresh=140, min_width=20):
    """
    Run the line-count heuristic along one wall segment.
    Returns two lists of (start, end, line_count) tuples:
      - window_candidates: cnt >= 3  (glass pane lines present)
      - door_candidates:   cnt <= 1  (outer face line absent or nearly gone)
    Segments < min_width pixels wide are discarded (corner junctions, noise).
    """
    if band['orientation'] == 'horizontal':
        scan_fn = lambda pos: count_dark_lines_in_col(
            gray, pos, band['row_top'], band['row_bot'], thresh)
        scan_range = range(band['scan_col_start'], band['scan_col_end'])
    else:
        scan_fn = lambda pos: count_dark_lines_in_row(
            gray, pos, band['scan_col_start'], band['scan_col_end'], thresh)
        scan_range = range(band['scan_row_start'], band['scan_row_end'])

    prev = 0; seg_start = scan_range.start; segments = []
    for pos in scan_range:
        cnt = scan_fn(pos)
        if cnt != prev:
            if prev > 0:
                segments.append((seg_start, pos - 1, prev))
            seg_start = pos
            prev = cnt
    if prev:
        segments.append((seg_start, scan_range.stop, prev))

    window_candidates = [(s, e, c) for (s, e, c) in segments
                         if c >= 3 and (e - s) >= min_width]
    door_candidates   = [(s, e, c) for (s, e, c) in segments
                         if c <= 1 and (e - s) >= min_width]
    return window_candidates, door_candidates

# Collect candidates across all walls
all_window_candidates = {}
all_door_candidates   = {}
for band in band_registry:
    wc, dc = find_candidates(gray, band)
    all_window_candidates[band['name']] = wc
    all_door_candidates[band['name']]   = dc
```

**Line-count signatures:**
- 2 lines → solid wall (discard)
- 3–4 lines → possible window (verify with zoom)
- 1 line → possible door gap — only inner face visible (verify with zoom)
- 0 lines → possible wide door gap or step corner (verify with zoom)
- 5+ lines → noise, junction, or furniture overlapping wall (discard)

---

## STEP 3 — Visual Confirmation with Zoomed Crops

For every candidate opening (window or door), save a zoomed crop before accepting it.
Use the band's face positions (from `band_registry`) to set the crop bounds —
do not estimate them from the image.

```python
from PIL import Image

def save_zoom_crop(img, band, cand_start, cand_end, label, pad=20):
    """Save a zoomed crop centred on one candidate opening."""
    if band['orientation'] == 'horizontal':
        x1 = cand_start - pad
        x2 = cand_end   + pad
        y1 = band['row_top']  - 10
        y2 = band['row_bot']  + 10
        scale_w, scale_h = 6, 8
    else:
        x1 = band['scan_col_start'] - 10
        x2 = band['scan_col_end']   + 10
        y1 = cand_start - pad
        y2 = cand_end   + pad
        scale_w, scale_h = 8, 4

    crop = img.crop((x1, y1, x2, y2))
    crop = crop.resize((crop.width * scale_w, crop.height * scale_h), Image.NEAREST)
    crop.save(f'zoom_{label}.png')

# Window candidates
for band in band_registry:
    for i, (start, end, cnt) in enumerate(all_window_candidates[band['name']]):
        save_zoom_crop(img, band, start, end, f"{band['name']}_win{i}")

# Door candidates
for band in band_registry:
    for i, (start, end, cnt) in enumerate(all_door_candidates[band['name']]):
        save_zoom_crop(img, band, start, end, f"{band['name']}_door{i}")
```

**Minimum zoom factor: 5× on the short axis, 3× on the long axis.**
Use `Image.NEAREST` (no interpolation) to preserve exact pixel shapes.

**What to look for — windows:**
- ✅ Both wall face lines intact across the full candidate width
- ✅ Two or three parallel grey lines between the face lines → glazing
- ✅ Small perpendicular rectangles at each end → jamb bumps
- ❌ Continuous thick band with no internal lines → solid wall (discard)

**What to look for — doors:**
- ✅ Outer face line **absent** (gap) across the full candidate width
- ✅ Quarter-circle arc visible in the interior zone → confirms door swing
- ✅ Inner face line may also be absent or thinned
- ⚠️  Outer face gap present but **no arc visible** → still treat as door, flag for user review
- ❌ Both face lines intact → not a door; re-examine as window candidate

---

## STEP 4 — Find the Exact Bump Positions

This is the most critical step. The bumps define the TRUE window boundary.
All scan zones below come from the band descriptor — no hardcoded pixel values.

### 4a. For horizontal walls (bumps are vertical notches)

Scan the **between-face zone** at each column near the suspected window edge.
A bump column shows dark pixels spanning the full wall band depth.

```python
def find_bump_col(gray, band, x_candidate, search_radius=30):
    """
    Find the outermost bump column near x_candidate on a horizontal wall.
    Scans the midpoint row of the glass pane zone.
    Returns the outermost dark column of the bump cluster, or None.
    """
    mid_row = (band['glass_row_start'] + band['glass_row_end']) // 2
    dark_cols = []
    for col in range(x_candidate - search_radius, x_candidate + search_radius):
        if gray[mid_row, col] < 160:
            dark_cols.append(col)

    if not dark_cols:
        return None

    # Find clusters (bumps are 3–6 consecutive dark pixels)
    clusters = []
    cluster = [dark_cols[0]]
    for c in dark_cols[1:]:
        if c == cluster[-1] + 1:
            cluster.append(c)
        else:
            clusters.append(cluster)
            cluster = [c]
    clusters.append(cluster)

    # Return the outermost column of the cluster closest to x_candidate
    closest = min(clusters, key=lambda cl: abs(sum(cl) / len(cl) - x_candidate))
    return min(closest)   # outermost = leftmost column of the bump
```

A bump produces a **cluster of 3–6 dark pixels** at consecutive columns.
The outermost dark column of that cluster = window boundary.

### 4b. For vertical walls (bumps are horizontal notches)

Scan each row in the glass pane zone across the full wall band width.
A bump row shows dark pixels spanning the FULL wall band width.

```python
def find_bump_row(gray, band, y_candidate, search_radius=20):
    """
    Find the outermost bump row near y_candidate on a vertical wall.
    Counts dark pixels across the full wall band width per row.
    Returns the outermost row of the bump cluster, or None.
    """
    col_start = band['scan_col_start']
    col_end   = band['scan_col_end']
    dark_rows = []
    for row in range(y_candidate - search_radius, y_candidate + search_radius):
        dark_count = sum(1 for c in range(col_start, col_end) if gray[row, c] < 160)
        if dark_count > 5:
            dark_rows.append(row)

    if not dark_rows:
        return None

    # Return the outermost row (closest to the exterior)
    outer_face = band['outer_face']
    return min(dark_rows, key=lambda r: abs(r - outer_face))
```

A bump row produces **8+ dark pixels** spanning the wall band.
The outermost such row = window boundary.

### 4c. Scan WIDER than you think necessary

The window symbol often extends beyond the nominal wall band on the interior side.
`build_band_registry` already adds 15px on both sides for vertical walls
(`scan_col_start/end`). For horizontal walls, bump search uses `search_radius=30`
by default. Increase these if bumps are still missed on a specific plan.

---

### 4d. For doors: find the gap edges in the outer face line

Do not use bump logic for doors — there are no bumps. Instead, find the exact
columns (or rows) where the outer face line **vanishes** and **reappears**. Those
are the true door boundaries.

```python
def find_door_gap_edges(gray, band, cand_start, cand_end,
                        search_pad=15, thresh=160):
    """
    Find the exact pixel positions where the outer face line disappears
    and reappears, defining the true door opening width.

    Returns (gap_start, gap_end) in the wall's long-axis coordinate,
    or None if the gap cannot be confirmed.
    """
    outer = band['outer_face']   # the row/col of the outer face line

    if band['orientation'] == 'horizontal':
        # Scan along the outer face ROW within the candidate column range
        scan_range = range(cand_start - search_pad, cand_end + search_pad)
        is_dark = lambda pos: gray[outer, pos] < thresh

        gap_start, gap_end = None, None
        in_gap = False
        for col in scan_range:
            dark = is_dark(col)
            if not dark and not in_gap:      # line just vanished → gap starts
                gap_start = col
                in_gap = True
            elif dark and in_gap:            # line just reappeared → gap ends
                gap_end = col - 1
                break

    else:  # vertical
        # Scan along the outer face COL within the candidate row range
        scan_range = range(cand_start - search_pad, cand_end + search_pad)
        is_dark = lambda pos: gray[pos, outer] < thresh

        gap_start, gap_end = None, None
        in_gap = False
        for row in scan_range:
            dark = is_dark(row)
            if not dark and not in_gap:
                gap_start = row
                in_gap = True
            elif dark and in_gap:
                gap_end = row - 1
                break

    if gap_start is None or gap_end is None:
        return None
    return gap_start, gap_end
```

**Usage — door gap edge detection:**

```python
confirmed_doors = []
for band in band_registry:
    for i, (start, end, cnt) in enumerate(all_door_candidates[band['name']]):
        result = find_door_gap_edges(gray, band, start, end)
        if result is None:
            print(f"  WARNING: could not confirm gap on {band['name']} door{i} "
                  f"— inspect zoom crop manually")
            continue
        gap_start, gap_end = result
        confirmed_doors.append({
            'name':        f"D{len(confirmed_doors)+1}",
            'wall':        band['name'],
            'orientation': band['orientation'],
            'gap_start':   gap_start,
            'gap_end':     gap_end,
            'band':        band,
        })
        print(f"  {band['name']} door{i}: gap [{gap_start} → {gap_end}]")
```

**What the gap edges give you:**
The gap boundaries are the jamb positions — exactly equivalent in purpose to the
bump positions for windows. Pass them to the same coordinate conversion in Step 6.

---

## STEP 5 — Verification Image

Draw all confirmed openings over the original plan in a **single image**.
Use two colours so windows and doors are immediately distinguishable at a glance:

- **Orange** `(255, 140, 0)` → windows  
- **Blue** `(30, 144, 255)` → doors

```python
from PIL import Image, ImageDraw

img_verify = Image.open('plan.png').convert('RGB')
draw = ImageDraw.Draw(img_verify)

ORANGE = (255, 140, 0)
BLUE   = (30, 144, 255)

def mark_opening(x1, y1, x2, y2, label, color):
    draw.rectangle([x1-2, y1-2, x2+2, y2+2], outline=color, width=3)
    if label:
        draw.rectangle([x1, y1-18, x1+len(label)*9+4, y1-2], fill=(255,255,255))
        draw.text((x1+3, y1-17), label, fill=color)

# Mark confirmed windows (orange)
for w in confirmed_windows:
    mark_opening(w['x1'], w['y1'], w['x2'], w['y2'], w['name'], ORANGE)

# Mark confirmed doors (blue)
for d in confirmed_doors:
    band = d['band']
    if band['orientation'] == 'horizontal':
        x1, x2 = d['gap_start'], d['gap_end']
        y1 = y2 = band['outer_face']
    else:
        y1, y2 = d['gap_start'], d['gap_end']
        x1 = x2 = band['outer_face']
    mark_opening(x1, y1, x2, y2, d['name'], BLUE)

img_verify.save('openings_verification.png')
```

**Do NOT proceed to Step 6 until the user has approved the verification image.**
All windows (orange) and doors (blue) must be reviewed together — corrections
to any opening should be made before converting to real-world coordinates.

---

## STEP 6 — Output: Handoff Data Block

`SCALE`, `OX`, and `OY` were loaded from the Phase 1 handoff at the top of this
skill — do not re-derive them. Emit a single unified `openings` list containing
both windows and doors, distinguished by a `type` field. The 3D modelling phase
uses `type` to apply the correct height defaults (see Architectural Defaults below).

```python
# SCALE, OX, OY already loaded from handoff in PHASE INPUT section

def px_to_m(col, row):
    """Convert pixel coords to plan-space metres (Y positive downward)."""
    x = round((col - OX) * SCALE, 3)
    y = round((row - OY) * SCALE, 3)
    return x, y

print("openings:")

# Windows
for w in confirmed_windows:
    x1, y1 = px_to_m(w['c1'], w['r1'])
    x2, y2 = px_to_m(w['c2'], w['r2'])
    print(f"  - name: {w['name']}")
    print(f"    type: window")
    print(f"    wall: {w['wall']}")
    print(f"    px:   [{w['c1']}, {w['r1']}] → [{w['c2']}, {w['r2']}]")
    print(f"    m:    x=[{x1}, {x2}]  y=[{y1}, {y2}]")

# Doors
for d in confirmed_doors:
    band = d['band']
    if band['orientation'] == 'horizontal':
        c1, r1 = d['gap_start'], band['outer_face']
        c2, r2 = d['gap_end'],   band['outer_face']
    else:
        c1, r1 = band['outer_face'], d['gap_start']
        c2, r2 = band['outer_face'], d['gap_end']
    x1, y1 = px_to_m(c1, r1)
    x2, y2 = px_to_m(c2, r2)
    print(f"  - name: {d['name']}")
    print(f"    type: door")
    print(f"    wall: {d['wall']}")
    print(f"    px:   [{c1}, {r1}] → [{c2}, {r2}]")
    print(f"    m:    x=[{x1}, {x2}]  y=[{y1}, {y2}]")
```

**Example output format:**
```yaml
openings:
  - name: W1
    type: window
    wall: top
    px:   [229, 60] → [328, 60]
    m:    x=[0.945, 2.430]  y=[0.000, 0.000]

  - name: W6
    type: window
    wall: bottom
    px:   [281, 754] → [498, 754]
    m:    x=[1.725, 4.980]  y=[10.350, 10.350]

  - name: D1
    type: door
    wall: bottom
    px:   [530, 758] → [614, 758]
    m:    x=[5.175, 6.435]  y=[10.500, 10.500]

  - name: W3
    type: window
    wall: right_step2_vertical
    px:   [1174, 143] → [1205, 249]
    m:    x=[15.120, 15.585]  y=[1.185, 2.775]
```

**Do NOT proceed to Blender.** Hand this data block to the
3D modelling phase (BLENDER_FLOORPLAN_3D_MODELING.md) for opening cutting.

---

## COMMON ERRORS AND FIXES

| Error | Symptom | Cause | Fix |
|---|---|---|---|
| Window too wide | Rect covers solid wall section | Using face-line transitions, not bump positions | Scan between-face zone for bump cluster |
| Window misses glazing on one side | Rect starts inside the symbol | Used inner bump face, not outer | Use outermost dark pixel of bump cluster |
| False positive on interior wall | Line count = 3 but no glazing visible in zoom | Interior partition near outer wall | Visually confirm with crop; if no parallel glass lines → solid wall |
| False positive at corner junction | Line count spikes at step corners | Two wall bands merging | Ignore segments < 20px wide; corners are never windows |
| Window on wrong wall | Detected glazing but on interior | Furniture or annotation near outer wall | Cross-check: window must be ON the outer perimeter line |
| Vertical wall window wrong width | Symbol extends beyond wall band | Scanned only nominal band cols | Extend scan 15px beyond expected interior face |
| Bump not found / wrong position | find_bump_* returns None or outlier | Phase 1 face row/col slightly off | Re-check the corresponding wall segment in the Phase 1 handoff before adjusting here |
| Door gap not confirmed | find_door_gap_edges returns None | Outer face line only faintly absent, or threshold too tight | Lower thresh in find_door_gap_edges to 180; inspect zoom crop |
| Door boundary too wide | Gap edges extend into solid wall | Outer face line has a worn pixel at the edge | Increase search_pad; visually trim to the last fully-absent position |
| Door classified as window | cnt == 2 for a door (inner face faint) | Inner face line nearly absent but not gone | In the zoom crop, check if the outer face line is intact; if gap → reclassify as door |
| Arc missing but gap confirmed | No arc visible in crop | Entrance door style — some architects omit the arc | Flag for user review but retain as door; do not discard |

---

## ARCHITECTURAL DEFAULTS (European)

| Element | Default Value |
|---|---|
| Window sill height | 0.90 m |
| Window height | 1.20 m |
| Door width | 0.80–1.00 m |
| Door height | 2.10 m |

---

## WORKFLOW SUMMARY

```
0. Load Phase 1 handoff YAML → SCALE, OX/OY, WALL_SEGMENTS
1. Build band_registry from WALL_SEGMENTS (face rows/cols, glass zones, scan bounds)
2. For each wall segment in band_registry:
     Run line-count heuristic within the segment's scan bounds
       → window_candidates (cnt >= 3)
       → door_candidates   (cnt <= 1)
3. For each candidate (window or door):
     Save a 5× zoomed crop and inspect visually
     Windows: confirm glazing lines + jamb bumps are present
     Doors:   confirm outer face line is absent; look for arc as secondary signal
4. WINDOW BRANCH — for each confirmed window:
     a. Horizontal walls: find_bump_col near each end of the candidate
     b. Vertical walls:   find_bump_row near each end of the candidate
     c. Use outermost dark pixel of each bump cluster as the boundary
     d. scan_col_start/end already extend 15px past interior face for vertical walls
   DOOR BRANCH — for each confirmed door:
     d. find_door_gap_edges: scan outer face row/col for where the line
        vanishes and reappears → those positions are the jamb edges
5. Generate ONE verification image:
     Orange rectangles = windows
     Blue rectangles   = doors
     Get user approval before proceeding
6. Convert all pixel bounds → plan-space metres using SCALE and OX/OY
   from the Phase 1 handoff (loaded in step 0)
7. Emit a unified YAML openings block with a `type: window|door` field per entry
8. STOP — hand off to the 3D modelling phase
```
