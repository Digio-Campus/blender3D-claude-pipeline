# SKILL: Outer Wall Extraction from Architectural Floor Plans
> Applicable to any 2D top-down architectural floor plan image.
> Designed to be followed by an AI agent without human assistance.

---

## GOAL
Extract the precise pixel coordinates of the **outer wall perimeter** of a building from a floor plan image, suitable for conversion to real-world units and 3D modeling.

---

## ⚠️ CRITICAL WARNINGS — READ BEFORE STARTING

### 1. The drawing border is NOT a wall
Floor plan images almost always have a 1-pixel (or very thin) rectangular border that frames the entire drawing sheet. This border is dark and will be detected as a "wall". It is NOT part of the building.

**Rule:** Always scan for a thin 1–3px dark border running the full width or height of the image at the very edges. Ignore it. The real outer walls will be a second, thicker band of dark pixels just inside that border.

### 2. A rectangle is rarely the right answer
Most buildings have notches, recesses, wings, or setbacks. **Never assume the perimeter is a simple rectangle.** Before building the polygon, always explicitly look for:
- Notches opening from the **top edge** (e.g. stair enclosures, light wells)
- Notches opening from the **bottom edge** (e.g. courtyards, service bays, rooms that step inward)
- Notches from the **left** or **right** edge
- L-shapes, T-shapes, U-shapes, E-shapes, etc.

**The user knows the building shape. Ask them to confirm the corner count before building the polygon.** A rectangle = 4 corners. Each rectangular notch or step adds 4 more corners (except cascading steps which add 2 per step after the first).

**Common shapes and corner counts:**
- Rectangle: 4 corners
- L-shape (one step on one side): 6 corners
- U-shape or rectangle with one notch: 8 corners
- Cascading/terraced (two steps in same direction): 8 corners
- Cascading/terraced (three steps in same direction): 10 corners
- Rectangle with two notches on different sides: 12 corners

**Cascading/stepped shapes** appear as a staircase on one side of the building outline. Do not confuse with interior staircases.

---

### 3. ALWAYS USE THE OUTER FACE — consistently for every coordinate

For all plans, **use the outer face of the wall band** for ALL coordinates: the four outer walls AND every notch wall AND every step wall. Outer face = `band_start` — the very first dark pixel encountered when scanning inward from the exterior.

**⚠️ Never mix outer-face coordinates with centre coordinates in the same polygon.** Use outer face exclusively for every single corner. Switching reference mid-polygon creates a lopsided, broken shape.

**Why outer face?** It is the single most unambiguous line in any floor plan — it is the clearest, outermost dark boundary and it requires no arithmetic to locate. The 3D modelling step uses it directly as the exterior wall surface and offsets inward by the full measured wall thickness to derive the interior face.

When measuring, always scan perpendicularly **at the midpoint of the segment** — never at an endpoint or corner, where two bands intersect and inflate each other's measurements.

**Outer face by wall direction:**
- Top wall (scan downward): outer face = topmost dark pixel = `band_start` (smallest Y)
- Bottom wall (scan upward): outer face = bottommost dark pixel = `band_start` (largest Y)
- Left wall (scan rightward): outer face = leftmost dark pixel = `band_start` (smallest X)
- Right wall (scan leftward): outer face = rightmost dark pixel = `band_start` (largest X)
- Notch side wall (left wall of notch): outer face = rightmost pixel of band (faces the notch space) = `band_end`
- Notch side wall (right wall of notch): outer face = leftmost pixel of band (faces the notch space) = `band_start`
- Notch inner wall (top notch): outer face = topmost pixel (faces the notch opening) = `band_start`
- Notch inner wall (bottom notch): outer face = bottommost pixel (faces the notch opening) = `band_end`

---

### 4. Dimension bracket lines are NOT wall faces

Plans with labeled dimensions draw **bracket tick lines** — thin (1–3px) dark marks at the exact position of the wall outer face, used as dimension annotation endpoints. They are NOT walls.

**Pattern:** scanning perpendicular to a wall you will see `1–3px dark` → `small gap (2–4px)` → `thick dark band (20–30px)`. The thin hit is the annotation bracket; the thick band is the real wall.

**Rule:** Require **≥ 20px** band thickness before accepting anything as an outer wall. Any band thinner than 20px is a bracket line, stair tread, annotation tick, or single-line partition — not an outer wall.

---

### 5. The true outer wall may NOT be the first thick band found from the edge

**⚠️ CRITICAL — hard-won lesson:** When scanning inward from an edge, the first thick band you hit is not always the outer wall of the building. The wrong band is commonly selected when:

- Sampling the **left or right** side at a y-position where an interior wall runs perpendicular to the outer wall and **merges** with it, making the merged blob look like the outermost band.
- Sampling the **bottom** at an x-position where a room-dividing wall hits the outer bottom wall and merges, inflating the band or appearing as a different depth.

**Safe method:** Sample each side at **multiple positions** spread across the full length. The consistent outermost band that appears at the majority of sample positions is the real outer wall. Use a median centre across all samples.

---

### 6. Architectural pillars are NOT wall corners

Floor plans frequently draw small filled black squares at wall junctions, wall terminations, and building corners. These are **structural pillars** and must not be confused with wall corners.

- A pillar is a compact dark square, isolated — it does not extend continuously in any direction for a significant distance.
- A wall is a long dark band, wide in one axis, narrow in the other, continuous for tens or hundreds of pixels.

**Rule:** Before accepting a corner coordinate, scan outward in both perpendicular directions for at least 50px. A real corner must have a continuous wall band in both directions. A pillar fails at least one check.

**The true outer wall ends at the near face of the pillar.** The corner coordinate is where the wall *meets* the pillar, not at the pillar's far edge.

---

### 7. Three drawing styles — and mixed conventions within the same plan

| Style | Wall appearance | Detection strategy | Threshold |
|---|---|---|---|
| **Thick-band** | Walls are solid dark-grey or black filled rectangles, 10–40px wide | Find the band centre by scanning perpendicularly | 100 |
| **Thin-line** | Walls are 1–3px dark outlines with light grey interior fill (~190–210) | Centre = midpoint of the two outline lines | 150 |
| **Single-line** | Walls are a single 1px dark line with no paired inner line | The line itself is the wall axis | 150 |

```python
# Pattern: DARK(1-3px) → WHITE(8-18px) → DARK(1-3px)  → thin-line style
# Pattern: DARK(10-40px)                                → thick-band style
# Pattern: DARK(1-3px) → WHITE continues               → single-line style
```

**⚠️ Mixed conventions are common.** A plan can use double-line for main outer walls and single-line for step/transition walls. **Check EACH wall segment independently.**

- Double-line wall: centre = `(outer_line_coord + inner_line_coord) // 2`
- Single-line wall: centre = the line coordinate itself
- Thick-band wall: centre = `(band_start + band_end) // 2`

---

### 8. External stairs, porches and ramps are NOT part of the perimeter

Entrance stairs and access ramps are open to the exterior — not enclosed rooms. The outer wall ends at the building face; the stairs hang outside it. In the flood fill image they appear **blue (exterior)**. Exclude them.

---

### 9. Diagonal walls require perpendicular scanning — NEVER row/column scanning alone

Some buildings have diagonal outer walls (e.g. a triangular entrance notch). These walls **cannot be reliably measured by simple horizontal row scans or vertical column scans alone**, because the wall band appears at a different position in each row/column and interior elements (stair treads, annotation lines) can easily contaminate those scans.

**The correct method for diagonal walls is:**

1. **Establish the slope first** using the flood fill image. Scan columns and check if `y_top` (the first dark pixel per column) increases or decreases at a consistent rate. A clean diagonal wall will show a perfectly consistent `+1/col` or `−1/col` rate with constant band width (±1–2px).

2. **Filter ruthlessly by band width.** Only accept columns where the full wall band width matches the expected wall thickness (e.g. 22–24px). Columns at junctions, stair crossings, or near other walls will show inflated or irregular widths — discard them.

3. **The true centre of a diagonal wall is the midpoint of the wall band measured perpendicularly.** For a 45° wall, this means:
   - If both faces move at `+1/col` → centre cy is constant (horizontal centre line)
   - If both faces move at `−1/col` → centre cy is constant (horizontal centre line)
   - If inner face is vertical and outer face moves at `+1/col` → centre moves at `+0.5/col`

4. **Two walls that are perpendicular to each other have perpendicular slopes:**
   - If wall A has slope `+1` (dy/dx = +1), wall B must have slope `−1` (dy/dx = −1) to be perpendicular to it.
   - Use this constraint to verify: if the right diagonal is confirmed at slope `+1`, the left diagonal forming a 90° angle with it MUST be slope `−1`.

5. **The centre line formula for a 45° wall:**
   - Slope +1: `cy = x + C` → find C from clean data points as `C = cy − x`
   - Slope −1: `cx = −y + C` → find C from clean data points as `C = cx + y`
   - C must be constant across all clean column/row measurements. If it varies by more than 2, you are measuring the wrong wall or a contaminated zone.

6. **Verify the formula against the flood fill image visually** by plotting the centre dots on the original plan image. Confirm each dot sits in the middle of the black wall band — not on a stair tread, not in white space.

7. **Find the apex** (where two diagonal walls meet) by solving the two centre line equations simultaneously:
   ```python
   # Right diagonal: cy = x + C_right
   # Left diagonal:  cx = -y + C_left  →  equivalently cy = -cx + C_left
   # Substitute: x + C_right = -x + C_left → 2x = C_left - C_right → x = (C_left - C_right) / 2
   cx_apex = (C_left - C_right) / 2
   cy_apex = cx_apex + C_right
   ```

8. **Extend the diagonal centre lines to the bottom (or top) wall** to find the base corners:
   ```python
   # Right diagonal base (where it meets bottom wall cy=bot_cy):
   cx_right_base = bot_cy - C_right
   # Left diagonal base:
   cx_left_base  = C_left - bot_cy
   ```

**⚠️ Do NOT trust row-scan midpoints for diagonal walls without first verifying band width consistency.** The most common error is picking up stair treads (which cross the diagonal wall zone) as the wall itself. Stair treads appear as thin (~3px) periodic lines and will not have consistent width across many consecutive rows.

**⚠️ Do NOT assume symmetry.** Even if the entrance looks symmetric, always measure both diagonals independently from the flood fill. One may have a different orientation or termination point than the other.

---

## STEP 0 — FLOOD FILL FIRST TO REVEAL THE TRUE BUILDING SHAPE

**Before any pixel measurement, always run a flood fill from a corner of the image.** This is the single most reliable step for understanding the building's shape without any guesswork.

```python
from PIL import Image
import numpy as np
from collections import deque

img = Image.open('plan.png').convert('RGB')
arr = np.array(img)
H, W = arr.shape[:2]

binary = (arr[:, :, 0] < 100).astype(np.uint8)  # 1 = wall, 0 = open

exterior = np.zeros((H, W), dtype=bool)
q = deque([(0, 0)])
exterior[0, 0] = True
while q:
    y, x = q.popleft()
    for dy, dx in [(-1,0),(1,0),(0,-1),(0,1)]:
        ny, nx = y+dy, x+dx
        if 0 <= ny < H and 0 <= nx < W and not exterior[ny,nx] and binary[ny,nx] == 0:
            exterior[ny, nx] = True
            q.append((ny, nx))

vis_arr = np.array(img.copy().convert('RGB'))
vis_arr[exterior] = [180, 200, 255]   # blue = exterior
vis_arr[binary == 1] = [0, 0, 0]      # black = walls
Image.fromarray(vis_arr).save('flood_fill.png')
```

**Study the flood fill image before doing anything else.** It shows you:
- Exactly how many notches the building has and from which sides
- Whether a staircase or porch is enclosed (white = interior → include) or open (blue = exterior → exclude)
- Which thick dark bands are outer walls vs. interior walls

---

## STEP 1 — Load Image and Establish Threshold

```python
from PIL import Image
import numpy as np

img = Image.open('plan.png').convert('RGB')
arr = np.array(img)

def is_wall(pixel, threshold=100):
    return int(pixel[0]) < threshold and int(pixel[1]) < threshold and int(pixel[2]) < threshold
```

**Use RGB, not grayscale, and cast to int before comparing** (avoids numpy uint8 overflow bugs).

**Threshold = 100** for thick-band plans. **Threshold = 150** for thin-line plans.

---

## STEP 2 — Detect Drawing Frame, Then Find the Four Outer Wall Bands

### 2a. Detect and discard the drawing frame
```python
MARGIN = 10  # ignore the first 10px from each edge
```

### 2b. Find the outermost real wall band on each side

**Sample at multiple positions across the full length of each side**, not just one. Use the position that is cleanest — away from corners and away from where interior walls meet the outer wall.

**Use min_thickness = 20** (not 10). Anything below 20px at a segment midpoint is a bracket, stair tread, or annotation — not an outer wall. The function must **skip thin bands and keep scanning** rather than returning the first dark band found.

```python
def find_outer_wall_band(arr, axis, sample_coord, threshold=100, min_thickness=20):
    """
    Scan inward along `axis` at position `sample_coord`.
    Returns (band_start, band_end, centre) of the first band >= min_thickness px thick.
    Skips thin bands (bracket lines, annotations) and keeps searching.

    axis: 'top' | 'bottom' | 'left' | 'right'
    sample_coord: x for top/bottom scans; y for left/right scans
    """
    H, W = arr.shape[:2]
    in_b = False
    bs = None

    if axis == 'top':
        scan = range(0, H // 2)
        get_px = lambda i: arr[i, sample_coord]
    elif axis == 'bottom':
        scan = range(H - 1, H // 2, -1)
        get_px = lambda i: arr[i, sample_coord]
    elif axis == 'left':
        scan = range(0, W // 2)
        get_px = lambda i: arr[sample_coord, i]
    else:  # right
        scan = range(W - 1, W // 2, -1)
        get_px = lambda i: arr[sample_coord, i]

    for i in scan:
        dark = int(get_px(i)[0]) < threshold
        if dark and not in_b:
            in_b = True
            bs = i
        elif not dark and in_b:
            thickness = abs(i - bs)
            if thickness >= min_thickness:
                lo, hi = min(bs, i - 1), max(bs, i - 1)
                outer_face = bs  # first dark pixel from exterior = outer face (consistent for all directions)
                return lo, hi, outer_face
            # Too thin — was an annotation line; reset and keep scanning
            in_b = False
            bs = None
    return None


def find_outer_wall_face_multi_sample(arr, axis, positions, threshold=100, min_thickness=20):
    """
    Sample the wall band at multiple positions. Returns the median outer face coordinate.
    Guards against interior walls merging with the outer band at some sample points.

    ⚠️ MERGE DETECTION: After collecting all outer face coords, compare each to the median.
    Any sample that returns a coord far from the median (> 30px away) has almost
    certainly merged with an interior wall at that position — discard it. The true
    outer wall is the one consistently present at the majority of sample positions,
    and the median naturally discards outliers from merge events.
    """
    outer_faces = []
    for pos in positions:
        result = find_outer_wall_band(arr, axis, pos, threshold, min_thickness)
        if result:
            outer_faces.append(result[2])
    if not outer_faces:
        return None
    outer_faces.sort()
    median = outer_faces[len(outer_faces) // 2]
    # Log any outliers for debugging — they indicate merge points to avoid
    outliers = [c for c in outer_faces if abs(c - median) > 30]
    if outliers:
        print(f"  [WARN] {axis} wall: {len(outliers)} merge outlier(s) discarded: {outliers}")
    return median
```

**Example — sample each side at 10+ evenly spaced positions:**
```python
H, W = arr.shape[:2]
x_positions = list(range(W // 10, W, W // 10))
y_positions = list(range(H // 10, H, H // 10))

top_outer_y    = find_outer_wall_face_multi_sample(arr, 'top',    x_positions)
bottom_outer_y = find_outer_wall_face_multi_sample(arr, 'bottom', x_positions)
left_outer_x   = find_outer_wall_face_multi_sample(arr, 'left',   y_positions)
right_outer_x  = find_outer_wall_face_multi_sample(arr, 'right',  y_positions)
```

---

## STEP 3 — Run Boundary Profiles on All Four Sides to Find Every Notch

**Never rely on a rectangle after finding the four main wall outer faces.** Profile every side to catch notches.

```python
def outermost_band_profile(arr, axis, scan_range, threshold=100, min_thickness=20, step=5):
    """
    For each position along a side, find the outer face coordinate of the outermost thick band.
    Returns list of (position, outer_face_coord_or_None).
    """
    profile = []
    for pos in range(scan_range[0], scan_range[1], step):
        result = find_outer_wall_band(arr, axis, pos, threshold, min_thickness)
        profile.append((pos, result[2] if result else None))
    return profile

def find_jumps(profile, jump_threshold=10):
    """Find positions where the outer face coord jumps by > jump_threshold px — each jump = a notch corner."""
    jumps = []
    prev_val = None
    for pos, val in profile:
        if prev_val is not None and val is not None and abs(val - prev_val) > jump_threshold:
            jumps.append((pos, prev_val, val))
        if val is not None:
            prev_val = val
    return jumps
```

**Run profiles on all four sides:**
```python
H, W = arr.shape[:2]
top_profile    = outermost_band_profile(arr, 'top',    (0, W))
bottom_profile = outermost_band_profile(arr, 'bottom', (0, W))
left_profile   = outermost_band_profile(arr, 'left',   (0, H))
right_profile  = outermost_band_profile(arr, 'right',  (0, H))

top_jumps    = find_jumps(top_profile)     # None values in profile = notch on top
bottom_jumps = find_jumps(bottom_profile)  # centre jumps upward = notch on bottom
# etc.
```

**Interpret results:**
- A region of `None` in the top/bottom profile = the outer wall is absent there → a notch opens upward or downward.
- A jump in the outer face value in the bottom/top profile (e.g. outer-face Y changes from 588 to 451) = the outer wall steps inward → a notch.
- Any jump ≥ 10px = a corner. Record its position and the two depth values flanking the jump.

**Notch detection rule — use this threshold:**
> Any group of **≥ 3 consecutive profile positions** where the outer face value differs by **≥ 40px** from the main wall outer face is a **notch candidate**. The x (or y) range of that group defines the notch opening. The new outer face value is the notch inner wall depth.
>
> Isolated single-position outliers (only 1–2 positions) are likely furniture, a thick interior partition touching the outer wall, or a scan artifact — not a notch.

**⚠️ A bottom notch where the wall steps inward is easy to miss** if you only sample the bottom at one x position where the outer wall is present. Always run the full profile.

---

## STEP 4 — Precisely Locate Each Notch Wall

Each notch has 4 corners: 2 at the outer wall level, 2 at the inner (closing) wall.

### 4a. Notch side walls (vertical walls bounding a top/bottom notch)

The notch side walls are **vertical** bands. Measure their band centre by scanning **horizontally** at a y-level well inside the notch (mid-depth), away from the top and inner walls where bands can merge.

```python
def find_notch_sidewall_outer_face(arr, approx_x, sample_y, notch_side, threshold=100,
                                    search_range=30, min_thickness=15):
    """
    Find the outer face coordinate of a vertical notch side wall by scanning horizontally.

    approx_x    : rough x position of the wall (from profile jump or visual inspection)
    sample_y    : a y row well inside the notch — NOT at the outer wall or inner wall level.
                  Use the midpoint between outer_wall_outer_face and notch_inner_outer_face.
    notch_side  : 'left'  — this is the LEFT side wall of the notch → outer face = band_end (rightmost pixel, faces notch space)
                  'right' — this is the RIGHT side wall of the notch → outer face = band_start (leftmost pixel, faces notch space)
    Returns     : (band_start_x, band_end_x, outer_face_x) or None

    Example band reading (left side wall):
        band_start=364, band_end=392, outer_face=392  ->  wall is 29px wide, outer face at x=392
    Example band reading (right side wall):
        band_start=614, band_end=642, outer_face=614  ->  wall is 29px wide, outer face at x=614
    """
    row = arr[sample_y]
    x_lo = max(0, approx_x - search_range)
    x_hi = min(arr.shape[1], approx_x + search_range)
    in_b = False; bs = None
    for x in range(x_lo, x_hi):
        dark = int(row[x][0]) < threshold
        if dark and not in_b:
            in_b = True; bs = x
        elif not dark and in_b:
            thickness = x - bs
            if thickness >= min_thickness:
                lo, hi = bs, x - 1
                outer_face = hi if notch_side == 'left' else lo
                return lo, hi, outer_face
            in_b = False; bs = None
    return None


# Usage example (top notch, left side wall):
#   outer_wall_outer_face_y = 84  (topmost pixel of top wall)
#   notch_inner_outer_face_y = 230  (topmost pixel of notch inner wall)
#   sample_y = (84 + 230) // 2 = 157  (well inside the notch, not at either wall)
sample_y = (outer_wall_outer_face_y + notch_inner_outer_face_y) // 2
result = find_notch_sidewall_outer_face(arr, approx_x=gap_edge_x, sample_y=sample_y, notch_side='left')
if result:
    lo, hi, notch_wall_outer_face_x = result
    print(f"  Notch left side wall: band x={lo}..{hi}, outer face x={notch_wall_outer_face_x}")
```

**Warning:** Do NOT sample at the outer wall cy or at the inner wall cy. At those y-levels the notch side wall merges with the horizontal wall band and the horizontal extent inflates the apparent band width, giving a wrong centre.

### 4b. Notch inner (closing) wall

```python
def find_notch_inner_wall_outer_face(arr, notch_x_start, notch_x_end, approx_depth_y,
                                      notch_opens_from, search=30, threshold=100):
    """
    Find the outer face coordinate of the horizontal wall closing the notch.

    notch_opens_from : 'top'    → notch opens from the top of the building
                                  → outer face = topmost pixel of inner wall (smallest Y, faces the notch opening)
                     : 'bottom' → notch opens from the bottom
                                  → outer face = bottommost pixel of inner wall (largest Y, faces the notch opening)
    """
    for y in range(approx_depth_y - search, approx_depth_y + search):
        if 0 <= y < arr.shape[0]:
            span = notch_x_end - notch_x_start
            dark_frac = sum(1 for x in range(notch_x_start, notch_x_end)
                            if int(arr[y, x][0]) < threshold) / span
            if dark_frac > 0.6:
                top = y
                while top > 0 and sum(1 for x in range(notch_x_start, notch_x_end)
                                       if int(arr[top-1,x][0]) < threshold) / span > 0.4:
                    top -= 1
                bot = y
                while bot < arr.shape[0]-1 and sum(1 for x in range(notch_x_start, notch_x_end)
                                                    if int(arr[bot+1,x][0]) < threshold) / span > 0.4:
                    bot += 1
                return top if notch_opens_from == 'top' else bot
    return approx_depth_y
```

---

## STEP 5 — Sample Wall Outer Faces at Midpoints (Not Endpoints)

For every segment, measure the outer face coordinate **at the midpoint of that segment**. Never at an endpoint or corner.

```python
# Horizontal wall segment: measure outer face Y at x = midpoint between its two endpoints
mid_x = (x_left_endpoint + x_right_endpoint) // 2
result = find_outer_wall_band(arr, 'top', mid_x)   # or 'bottom'
# result = (band_start, band_end, outer_face)
# e.g. (84, 112, 84)  ->  band is 29px wide, outer face y = 84  (topmost pixel)
wall_outer_y = result[2]

# Vertical wall segment: measure outer face X at y = midpoint between its two endpoints
mid_y = (y_top_endpoint + y_bottom_endpoint) // 2
result = find_outer_wall_band(arr, 'left', mid_y)  # or 'right'
# e.g. (195, 223, 195)  ->  band is 29px wide, outer face x = 195  (leftmost pixel)
wall_outer_x = result[2]
```

**Reading the result tuple:** `(band_start, band_end, outer_face)`. The outer face is always `band_start` — the first dark pixel encountered when scanning inward from the exterior. Record `band_start` and `band_end` alongside the outer face — they let you verify the wall thickness is consistent across all segments (they should all agree to within ±2px on a well-drawn plan).

**Wall thickness from the band:** `wall_thickness_px = band_end - band_start + 1`. Derive this from at least one measurement and include it in the output as `wall_thickness_px` and `wall_thickness_m`.

**After measuring each outer face:** confirm the pixel at `outer_face` is dark. If not, the measurement is off — re-measure.

---

## STEP 6 — Handle Stairs and Windows

**Stairs** appear as 3+ equally-spaced parallel dark lines. The outer wall is **continuous** through a stair region — bridge over it.

**External entrance stairs** below the building appear as a stair tread pattern outside the outer bottom wall. In the flood fill they are blue (exterior). The perimeter polygon ends at the outer bottom wall — do NOT extend to include the stairs.

**Windows and doors:** bridge over them — the structural wall perimeter continues straight through all openings.

---

## STEP 6b — Handle Diagonal Walls

If the flood fill reveals diagonal outer walls (e.g. a triangular entrance notch), follow this procedure instead of the standard perpendicular scan.

### 1. Determine slope from flood fill wall pixels

Scan columns across the diagonal zone. For each column x, find the first and last wall pixel in the expected row range. Accept only columns where `band_width = y_bot − y_top + 1` is within ±2px of the known wall thickness.

```python
clean_pts = []
for x in range(x_start, x_end):
    col = wall_mask[y_min:y_max, x]
    ys = np.where(col)[0] + y_min
    if len(ys) >= 3:
        y1, y2 = ys[0], ys[-1]
        width = y2 - y1 + 1
        if abs(width - expected_thickness) <= 2:
            clean_pts.append((x, y1, y2, (y1+y2)/2))
```

### 2. Fit and verify the centre line

```python
xs  = np.array([p[0] for p in clean_pts])
cys = np.array([p[3] for p in clean_pts])
slope, intercept = np.polyfit(xs, cys, 1)
# slope should be very close to +1.0 or −1.0 for a 45° wall
# For slope +1: cy = x + C  →  C = cy - x  (constant for all clean pts)
# For slope -1: cy = -x + C →  C = cy + x  (constant for all clean pts)
```

**Verify:** compute `C = cy − x` or `C = cy + x` for every clean point. All values must agree within ±2px. If they vary more, you have contaminated data — narrow the x range to exclude junction zones and stair crossings.

### 3. Solve for the apex

```python
# Right diagonal: cy = x + C_r   →  at a given y: cx = y - C_r
# Left  diagonal: cx = C_l - y   →  at a given y: cx = C_l - y
# Apex where both cx values are equal:
# y - C_r = C_l - y  →  2y = C_l + C_r  →  y_apex = (C_l + C_r) / 2
y_apex  = (C_left + C_right) / 2
cx_apex = y_apex - C_right   # same as C_left - y_apex
```

### 4. Extend to the bottom (or top) wall

```python
# Base of right diagonal where it meets bottom wall at y = bot_cy:
cx_right_base = bot_cy - C_right
# Base of left diagonal where it meets bottom wall at y = bot_cy:
cx_left_base  = C_left - bot_cy
```

### 5. Always verify visually

Plot the computed centre dots on the original plan image and confirm each dot sits **inside the black wall band** — not on a stair tread, not on a bracket line, not in white space. This visual check is mandatory; numerical data alone is not sufficient because stair treads and other diagonal elements can produce plausible but wrong fits.

**⚠️ Key pitfall:** Two seemingly consistent data groups with opposite slopes may appear in the same scan zone. One will be stair treads (thin, ~3px width, periodic), the other the real wall (consistent 20–26px width). Always check band width to distinguish them.

---

## STEP 7 — Build and Validate the Closed Perimeter Polygon

Assemble corners in clockwise order (Y increasing downward). At each corner, one coordinate stays constant (the outer face of the wall it sits on) and the other changes (the outer face of the intersecting wall).

```python
# Example: 12-corner building with top notch + bottom notch
# All coordinates are OUTER FACE values — the exterior boundary of the building.
perimeter_px = [
    (left_outer_x,           top_outer_y),              # 1  TL
    (top_notch_l_outer_x,    top_outer_y),              # 2  top notch opens   (notch left wall outer face = right pixel of band)
    (top_notch_l_outer_x,    top_notch_inner_outer_y),  # 3  top notch inner-left  (inner wall outer face = topmost pixel)
    (top_notch_r_outer_x,    top_notch_inner_outer_y),  # 4  top notch inner-right
    (top_notch_r_outer_x,    top_outer_y),              # 5  top notch closes  (notch right wall outer face = left pixel of band)
    (right_outer_x,          top_outer_y),              # 6  TR
    (right_outer_x,          bottom_outer_y),           # 7  BR
    (bot_notch_r_outer_x,    bottom_outer_y),           # 8  bottom notch opens (right side)
    (bot_notch_r_outer_x,    bot_notch_inner_outer_y),  # 9  bottom notch inner-right (inner wall outer face = bottommost pixel)
    (bot_notch_l_outer_x,    bot_notch_inner_outer_y),  # 10 bottom notch inner-left
    (bot_notch_l_outer_x,    bottom_outer_y),           # 11 bottom notch closes (left side)
    (left_outer_x,           bottom_outer_y),           # 12 BL
    (left_outer_x,           top_outer_y),              # close
]
```

**Validation checklist:**
- [ ] Corner count matches what the user confirmed
- [ ] Every turn is exactly 90° (one coordinate of adjacent points is shared)
- [ ] Polygon is closed (first == last point)
- [ ] No segment passes through an exterior opening or exterior space (check flood fill)
- [ ] All outer face values were measured at segment midpoints, not endpoints
- [ ] Every coordinate uses outer face measurements — never centres, never inner faces

---

## STEP 8 — Convert Pixels to Real-World Units

```python
scale_m_per_px = real_dimension_m / pixel_span

# pixel_span = distance between wall outer faces (outer face to outer face = full building extent)
# Origin at TL outer corner, Y negative downward (Blender convention)
origin_x, origin_y = perimeter_px[0]
perimeter_m = [
    (round((x - origin_x) * scale_m_per_px, 3),
     round(-(y - origin_y) * scale_m_per_px, 3))
    for (x, y) in perimeter_px
]
```

**If NO dimensions are labeled**, do NOT guess. Tell the user: *"No scale reference found. Please provide a building dimension."* Leave `scale_m_per_px` as null.

**Note:** When handing off to Blender, Y must be negated. Map pixel-Y → negative Blender-Y.

---

## STEP 9 — Verify with Visualization

Always generate a verification image before proceeding to 3D:

```python
from PIL import ImageDraw
img_out = img.copy().convert('RGB')
draw = ImageDraw.Draw(img_out)

for i in range(len(perimeter_px) - 1):
    draw.line([perimeter_px[i], perimeter_px[i+1]], fill=(0, 200, 0), width=4)

for i, (px, py) in enumerate(perimeter_px[:-1]):
    r = 12
    draw.ellipse([px-r, py-r, px+r, py+r], fill=(255, 0, 0))
    draw.text((px+14, py-8), str(i+1), fill=(255, 0, 0))

img_out.save('verification.png')
```

**Checklist — examine every side carefully:**
- [ ] Green line sits **on the outer face** of the black wall band on all sides — at the outermost dark pixel, not in the centre, not outside
- [ ] All notch corners sit on the outer face of their wall bands (red dot on the outermost dark pixel)
- [ ] No segment cuts across empty white interior space
- [ ] No segment extends outside the building
- [ ] Stair regions are bridged (straight line, no jog)
- [ ] Window and door gaps are bridged
- [ ] External stairs/porches are excluded
- [ ] Polygon fully closed

**Do NOT proceed to Blender until the user has approved the verification image.**

---

## STEP 10 — Pixel-Level Validation of Each Corner

```python
def validate_corner(arr, px, py, radius=8, threshold=100):
    """Returns True if >50% of pixels within radius are dark (corner is on a wall)."""
    checks = []
    for dy in range(-radius, radius+1):
        for dx in range(-radius, radius+1):
            y, x = py+dy, px+dx
            if 0 <= y < arr.shape[0] and 0 <= x < arr.shape[1]:
                checks.append(int(arr[y, x][0]) < threshold)
    return sum(checks) / len(checks) > 0.5
```

If a corner fails, it is floating in white space — re-measure that coordinate.

---

## COMMON PITFALLS AND FIXES

| Problem | Cause | Fix |
|---|---|---|
| Entire left (or right) wall identified at the wrong position | Sampled at a y-level where an interior wall merges with the outer wall — the merged blob was mistaken for the outer wall, which was actually further left/right | Sample at **multiple y positions** across the full height; take the median. The true outer wall is the one consistently present at nearly all sample positions. |
| Bottom notch completely missed | Only sampled bottom at one or two x positions where the outer bottom wall is flat — never detected the inward step | Run `outermost_band_profile` on the full bottom at every 5px. Any jump ≥ 10px in the outer face value = a notch. |
| Outer wall identified as interior wall | First thick band from the edge was an interior wall, not the outer wall | Use `find_outer_wall_band` with skip-thin-band logic (min_thickness=20, reset on thin band and keep scanning); cross-check with flood fill. |
| Coordinates mix outer-face and centre measurements | Different steps used different reference systems | Re-measure every coordinate using `find_outer_wall_band(...)[2]`. Outer face throughout — never mix. |
| Dimension bracket line detected as outer wall face | 1px annotation tick at wall outer face position looks like a wall | Require min_thickness ≥ 20px; 1–3px bands are never outer walls. |
| Building assumed rectangular | Skipped boundary profiling | Run `outermost_band_profile` on all 4 sides immediately after finding the four main wall outer faces. |
| Drawing border detected as outer wall | 1px frame of the sheet | Ignore first 10px from image edge. |
| Notch depth wrong | Only checked one y level | Use `find_notch_inner_wall_outer_face` — scan across the full notch width to find the closing wall. |
| External stairs included in perimeter | Stair box below building looks like a notch | Check flood fill — blue = exterior → not part of perimeter. |
| Stair gap treated as notch | Saw a gap in wall scan | Check for repeating parallel lines ~10–15px apart; if present, it is stair treads → bridge the gap. |
| Corner on white space | Endpoint coordinates used instead of midpoint-sampled outer faces | Always measure outer faces by scanning perpendicularly at midpoints of each segment. |
| Wrong wall thickness | Sampled at a corner where two bands merge | Sample at midpoint of segment, well away from corners and T-junctions. |
| Notch side wall outer face on wrong side | Used band_start for left notch side wall instead of band_end | Left-side notch wall → outer face = band_end (faces right, into notch). Right-side notch wall → outer face = band_start (faces left, into notch). Use `find_notch_sidewall_outer_face` with correct `notch_side` parameter. |
| Interior partition mistaken for outer wall inner face | Parallel interior wall at same distance as expected inner face | Check continuity ≥ 80% of wall length; partition appears only in a section. |
| Step/transition walls measured as wrong style | Assumed all walls have same drawing style | Cross-section scan EACH wall segment independently. |
| Architectural pillar mistaken for wall corner | Filled black squares appear at wall terminations | Verify by scanning both perpendicular axes ≥ 50px — real corners have continuous walls in both directions. |
| Isolated dark segments from pillar bases | Pillar bases create short bands that look like wall segments | Verify vertical continuity ≥ 50px; no wall behind it = pillar base. |
| Anti-aliased wall edge missed | Threshold too strict | Use threshold = 100 to capture anti-aliased edge pixels. |
| No dark walls found (walls ~195 grey) | Thin-line plan; grey fill mistaken for walls | Switch to threshold=150; look for 1–3px darker outlines at wall edges. |
| Scale unknown | No dimensions labeled on plan | Ask the user for a reference measurement; do not guess. |
| Diagonal wall outer face identified incorrectly | Used row/column scan midpoints without filtering by band width; stair treads contaminated the scan | Filter to only columns/rows with band width matching expected wall thickness (±2px). Verify by plotting outer face dots on the plan image. |
| Two contradictory data groups found for diagonal wall | One group is stair treads, the other is the real wall | Stair treads appear as thin (~3px) periodic bands; real wall has consistent 20–26px width across a long continuous run. |
| Diagonal walls assumed symmetric | Relied on mirroring one diagonal to the other | Always measure both diagonals independently from the flood fill. |
| Diagonal wall slope estimated incorrectly | Used data points near a junction or stair-crossing zone | Use only the clean middle section; slope must be perfectly consistent (+1 or −1) across ≥10 consecutive columns. |
| Wrong apex position | Centre line formulae not solved simultaneously | Solve both centre line equations simultaneously; verify the apex pixel sits on dark wall pixels in the flood fill. |

---

## OUTPUT FORMAT (for 3D modeling handoff)

```yaml
scale_m_per_px: 0.02143
wall_thickness_px: 29
wall_thickness_m: 0.62
corner_count: 12
perimeter_px:
  - { x:   209, y:   98 }
  - { x:   378, y:   98 }
  - { x:   378, y:  243 }
  - { x:   513, y:  243 }
  - { x:   513, y:   98 }
  - { x:   956, y:   98 }
  - { x:   956, y:  588 }
  - { x:   856, y:  588 }
  - { x:   856, y:  451 }
  - { x:   614, y:  451 }
  - { x:   614, y:  588 }
  - { x:   209, y:  588 }
  - { x:   209, y:   98 }
perimeter_m:
  - { x:  0.000, y:   0.000 }
  - { x:  3.621, y:   0.000 }
  - { x:  3.621, y:  -3.107 }
  - { x:  6.514, y:  -3.107 }
  - { x:  6.514, y:   0.000 }
  - { x: 16.007, y:   0.000 }
  - { x: 16.007, y: -10.500 }
  - { x: 13.864, y: -10.500 }
  - { x: 13.864, y:  -7.564 }
  - { x:  8.679, y:  -7.564 }
  - { x:  8.679, y: -10.500 }
  - { x:  0.000, y: -10.500 }
  - { x:  0.000, y:   0.000 }
```

**Note on coordinate system:** `perimeter_m` uses origin at top-left outer corner, X positive right, Y negative downward (to match Blender's Y-forward convention after flipping). All coordinates represent the **outer face** of the building perimeter.

**Computing wall_thickness_m:** Always derive this explicitly — do not leave it as a placeholder:
```python
# wall_thickness_px = band_end - band_start + 1  (measure on any outer wall at its midpoint)
# e.g. band (195, 223) -> thickness = 223 - 195 + 1 = 29px
wall_thickness_m = round(wall_thickness_px * scale_m_per_px, 3)
# This value is passed to the 3D modelling step and used as T (full wall thickness).
# The modelling step uses the outer face polygon directly as the exterior surface,
# then offsets inward by T to derive the interior face.
# A forgotten or wrong value here silently produces the wrong wall geometry.
```

**The example above is a verified real extraction (plano6.png):** a 16.0 × 10.5 m single-storey house with:
- A **top notch** (corridor/stair recess opening from the top wall)
- A **bottom notch** (rooms stepping inward in the lower-centre section)
- An external entrance staircase below the bottom wall, correctly **excluded**
- All coordinates are **outer face** values — e.g. top-left corner at x=209 is the leftmost pixel of the left wall band, y=98 is the topmost pixel of the top wall band
- The true left outer wall at x=209, significantly further left than several early attempts that anchored to an interior wall band at x=364–378 due to merging at certain sample positions
