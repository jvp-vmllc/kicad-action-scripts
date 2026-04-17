---
title: KiCad 10 Port — What Changed
tags:
  - kicad
  - kicad10
  - via-stitching
  - changelog
created: 2026-04-17
status: done
---

# KiCad 10 Port — What Changed

> [!info] Parent note
> [[kicad-action-scripts-kicad10]]

Commit `34b55b9` on branch `pr94` — based on upstream [PR #94](https://github.com/jsreynaud/kicad-action-scripts/pull/94).

Three files were modified, all inside `ViaStitching/`.

---

## FillArea.py

### 1. `VIATYPE_THROUGH` removed from Python bindings

KiCad 10 dropped this constant from its SWIG bindings entirely.

```python
# Added at module level
try:
    _ = VIATYPE_THROUGH
except NameError:
    VIATYPE_THROUGH = 3  # VIATYPE::THROUGH integer value
```

---

### 2. Version string comparison was broken

`Version()` returns `"10.0.0"` in KiCad 10. The old guard `Version() < "7"` uses **lexicographic** comparison, so `"10.0.0" < "7"` evaluates to `True` — KiCad 10 was being treated as older than KiCad 7.

```python
# Added helper
def _kicad_version_major():
    try:
        return int(Version().split('.')[0])
    except Exception:
        return 0

# All occurrences replaced:
# Before:  if Version() < "7":
# After:   if _kicad_version_major() < 7:
```

> [!warning] This was a silent bug — the wrong code path ran without any error on KiCad 10.

---

### 3. `HitTestInsideZone` removed

KiCad 10 removed `HitTestInsideZone` from `ZONE`. The old code used a bare `except` that swallowed the error and **silently disabled** keepout zone checking.

```python
# Before — silent failure
try:
    hit_test_zone = area.HitTestInsideZone(point_to_test)
except:
    hit_test_zone = False  # keepout logic quietly skipped

# After — explicit fallback
if hasattr(area, 'HitTestInsideZone'):
    hit_test_zone = area.HitTestInsideZone(point_to_test)
else:
    hit_test_zone = area.HitTest(point_to_test)  # KiCad 10+
```

The same `hasattr` pattern is applied in the zone priority check lambda:

```python
_hittest_inside = (
    (lambda z, pt: z.HitTestInsideZone(pt))
    if hasattr(area, 'HitTestInsideZone')
    else (lambda z, pt: z.HitTest(pt))
)
```

---

### 4. `GetPriority()` renamed to `GetAssignedPriority()`

```python
# Before
lambda x: x.GetPriority() > area_priority

# After
lambda x: x.GetAssignedPriority() > area_priority
```

---

### 5. `GetBoardPolygonOutlines()` — new required parameter

KiCad 10 made `aInferOutlineIfNecessary` mandatory. Handled with try/except for backward compatibility:

```python
try:
    self.pcb.GetBoardPolygonOutlines(board_edge)
except TypeError:
    self.pcb.GetBoardPolygonOutlines(board_edge, True)  # KiCad 10
```

---

### 6. `ConcentricFillVias()` rewritten

The old per-zone loop had cross-layer polygon merge artefacts and relied on `Version() < "7"` (broken — see #2 above). Rewritten to build a **single unified polygon intersection** across all enabled copper layers first, then place vias along it:

```python
poly_set = None
for layer_id in self.pcb.GetEnabledLayers().CuStack():
    poly_set_layer = SHAPE_POLY_SET()
    for zone in zones:
        if zone.IsOnLayer(layer_id):
            poly_set_layer.Append(zone.Outline())
    if poly_set is None:
        poly_set = poly_set_layer
    else:
        poly_set.BooleanIntersection(poly_set_layer)
        poly_set.Simplify()
```

---

### 7. Grid origin system removed

The `GRID_TYPE_BOARD_BOUNDS / ABSOLUTE / GRID_ORIGIN` system was removed. The grid origin now always uses the board's own position — this was already the default and the other options caused edge-case bugs.

```python
# Before
origin = self.GetGridOrig(lboard)   # depended on self.grid_type

# After
origin = lboard.GetPosition()       # always board position
```

Grid extent variables `x_min`, `y_min`, `x_max`, `y_max` and all their uses in coordinate calculations were also simplified away.

### 8. `SetFillType()` renamed to `SetType()`

```python
def SetType(self, type):
    self.fill_type = type
    return self
```

---

## FillAreaAction.py

- Removed `fill.SetGridType(a.m_cbGridType.GetStringSelection())` — Grid Origin UI is gone
- Changed `fill.SetFillType(...)` → `fill.SetType(...)`

---

## FillAreaDialog.py

| What | Before | After |
|---|---|---|
| Dialog height | `663` px | `590` px |
| `m_SizeMM` min width | `SetMinSize(wx.Size(1000, -1))` — pushed Pattern off-screen | Removed |
| Grid Origin combo | Present (`m_cbGridType`) | Removed |
| Help bitmap | `wx.Bitmap("stitching-vias-help.png", ...)` (relative path, broken) | `wx.NullBitmap` (loaded at runtime by `FillAreaAction.py`) |

---

## See also

- [[kicad10-api-reference]] — quick-reference card of all changed APIs
- [[kicad10-build-guide]] — how to rebuild the PCM zip
