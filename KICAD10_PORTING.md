# KiCad 10 Compatibility Port

This document describes the changes made to port `kicad-action-scripts` to KiCad 10's Python API, and how to build the installable `.zip` package for KiCad's Plugin and Content Manager (PCM).

---

## What Changed

### Commit: `34b55b9` — `fix(ViaStitching): port to KiCad 10 Python API`

Applied the fixes from upstream PR [jsreynaud/kicad-action-scripts#94](https://github.com/jsreynaud/kicad-action-scripts/pull/94) across three files in `ViaStitching/`.

---

### `ViaStitching/FillArea.py`

#### 1. `VIATYPE_THROUGH` fallback

KiCad 10 removed `VIATYPE_THROUGH` from its Python bindings. A try/except guard defines it manually if missing:

```python
try:
    _ = VIATYPE_THROUGH
except NameError:
    VIATYPE_THROUGH = 3  # VIATYPE::THROUGH integer value
```

#### 2. Version detection helper — `_kicad_version_major()`

The old code used string comparison (`Version() < "7"`), which breaks on KiCad 10 because `"10.0.0" < "7"` is `True` lexicographically. Replaced with an integer comparison:

```python
def _kicad_version_major():
    try:
        return int(Version().split('.')[0])
    except Exception:
        return 0
```

All `Version() < "7"` checks replaced with `_kicad_version_major() < 7`.

#### 3. `HitTestInsideZone` — replaced bare `except` with `hasattr()`

KiCad 10 removed `HitTestInsideZone`. Old code silently swallowed the exception and skipped keepout logic entirely. Now uses `hasattr()` and falls back to `HitTest()`:

```python
if hasattr(area, 'HitTestInsideZone'):
    hit_test_zone = area.HitTestInsideZone(point_to_test)
else:
    hit_test_zone = area.HitTest(point_to_test)  # KiCad 10+
```

#### 4. `GetPriority()` → `GetAssignedPriority()`

KiCad 10 renamed the zone priority method:

```python
# Before
x.GetPriority() > area_priority
# After
x.GetAssignedPriority() > area_priority
```

#### 5. `GetBoardPolygonOutlines()` — new required parameter in KiCad 10

KiCad 10 made `aInferOutlineIfNecessary` a required argument. Wrapped in try/except for backward compatibility:

```python
try:
    self.pcb.GetBoardPolygonOutlines(board_edge)
except TypeError:
    self.pcb.GetBoardPolygonOutlines(board_edge, True)  # KiCad 10
```

#### 6. `ConcentricFillVias()` — rewritten

The previous per-zone loop had issues with cross-layer polygon merging. Rewritten to compute a single polygon intersection across all enabled copper layers upfront, then place vias along that unified set:

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

#### 7. Grid origin simplified

Removed `GRID_TYPE_BOARD_BOUNDS / ABSOLUTE / GRID_ORIGIN` system. Origin now always uses board position:

```python
# Before
origin = self.GetGridOrig(lboard)

# After
origin = lboard.GetPosition()
```

Grid extent calculations (`x_min`, `y_min`, `x_max`, `y_max`) also simplified accordingly.

#### 8. `SetFillType()` renamed to `SetType()`

```python
def SetType(self, type):
    self.fill_type = type
    return self
```

---

### `ViaStitching/FillAreaAction.py`

- Removed `fill.SetGridType(...)` call (Grid Origin UI removed)
- Updated `fill.SetFillType(...)` → `fill.SetType(...)`

---

### `ViaStitching/FillAreaDialog.py`

- Removed **Grid Origin** combo box (`m_cbGridType`) and its label — no longer needed
- Removed `SetMinSize(wx.Size(1000, -1))` on `m_SizeMM` — was pushing the Pattern dropdown off-screen
- Changed dialog initial height: `663` → `590`
- Changed `wx.Bitmap("stitching-vias-help.png", ...)` → `wx.NullBitmap` (bitmap is loaded at runtime in `FillAreaAction.py` using the correct absolute path)

---

## How to Build the PCM Package

The KiCad Plugin and Content Manager expects a `.zip` with this layout:

```
plugins/
  __init__.py
  ViaStitching/
    FillArea.py
    FillAreaAction.py
    FillAreaDialog.py
    __init__.py
    FillAreaTpl.fbp
    stitching-vias.png
    stitching-vias.svg
    stitching-vias-help.png
    stitching-vias-help.svg
  CircularZone/
    CircularZone.py
    CircularZoneDlg.py
    CircularZoneDlg.fbp
    __init__.py
    round_keepout_area.png
    round_keepout_area.svg
resources/
  icon.png
metadata.json
```

### Prerequisites

- Python 3 (standard library only — `zipfile`, `json`, `os`)
- A copy of the previous release zip (used to pull unchanged `CircularZone` and `resources/` assets):
  ```
  https://github.com/jsreynaud/kicad-action-scripts/releases/download/v9.0-0.7.0/kicad-action-scripts_v9.0-0.7.0.zip
  ```

### Build script

Save this as `build_pcm.py` next to the repo, then run `python build_pcm.py`:

```python
import zipfile, os, json

REPO    = r"path/to/kicad-action-scripts"   # repo root
REF_ZIP = r"path/to/kicad-action-scripts_v9.0-0.7.0.zip"
OUT     = r"path/to/kicad-action-scripts_v10.0-0.7.0.zip"

VIA_FILES = [
    "FillArea.py", "FillAreaAction.py", "FillAreaDialog.py",
    "__init__.py", "FillAreaTpl.fbp",
    "stitching-vias.png", "stitching-vias.svg",
    "stitching-vias-help.png", "stitching-vias-help.svg",
]
CIRCULAR_FILES = [
    "CircularZone.py", "__init__.py", "CircularZoneDlg.py", "CircularZoneDlg.fbp",
    "round_keepout_area.png", "round_keepout_area.svg",
]
METADATA = {
    "$schema": "https://go.kicad.org/pcm/schemas/v1",
    "name": "kicad-action-scripts",
    "description": "ViaStitching and more tools",
    "description_full": "Via stitching action tool and circular zone builder action tool.",
    "identifier": "com.github.jsreynaud.kicad-action-scripts",
    "type": "plugin",
    "author": {"name": "jsreynaud", "contact": {"email": "js.reynaud@gmail.com"}},
    "license": "GPL-2.0",
    "resources": {
        "Github": "https://github.com/jsreynaud/kicad-action-scripts",
        "Readme": "https://github.com/jsreynaud/kicad-action-scripts/blob/master/README.md"
    },
    "tags": ["annular", "pcbnew", "dxf", "grid", "mcad"],
    "versions": [{
        "version": "10.0.0",
        "status": "stable",
        "kicad_version": "10.0",
        "platforms": ["linux", "macos", "windows"]
    }]
}

with zipfile.ZipFile(OUT, "w", zipfile.ZIP_DEFLATED) as zout:
    # Modified ViaStitching files from the repo
    for fname in VIA_FILES:
        src = os.path.join(REPO, "ViaStitching", fname)
        zout.write(src, f"plugins/ViaStitching/{fname}")

    # Unchanged CircularZone + shared assets from previous release zip
    with zipfile.ZipFile(REF_ZIP, "r") as zref:
        for fname in CIRCULAR_FILES:
            key = f"plugins/CircularZone/{fname}"
            zout.writestr(key, zref.read(key))
        zout.writestr("plugins/__init__.py", zref.read("plugins/__init__.py"))
        zout.writestr("resources/icon.png",  zref.read("resources/icon.png"))

    zout.writestr("metadata.json", json.dumps(METADATA, indent=2))

print(f"Package written to: {OUT}")
```

### Install in KiCad

1. Open KiCad → **Plugin and Content Manager**
2. Click **Install from File…**
3. Select `kicad-action-scripts_v10.0-0.7.0.zip`
4. Restart KiCad if prompted

---

## Branch / Release

| Item | Value |
|---|---|
| Fork | [jvp-vmllc/kicad-action-scripts](https://github.com/jvp-vmllc/kicad-action-scripts) |
| Branch | `pr94` |
| Based on upstream PR | [jsreynaud/kicad-action-scripts#94](https://github.com/jsreynaud/kicad-action-scripts/pull/94) |
| Package | `kicad-action-scripts_v10.0-0.7.0.zip` |
