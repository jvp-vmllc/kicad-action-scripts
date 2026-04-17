---
title: KiCad 10 Python API — Breaking Changes Reference
tags:
  - kicad
  - kicad10
  - api
  - python
  - reference
created: 2026-04-17
status: done
---

# KiCad 10 Python API — Breaking Changes Reference

> [!info] Parent note
> [[kicad-action-scripts-kicad10]]

Quick-reference card for every KiCad Python API that broke between KiCad 9 and KiCad 10, as encountered during the `kicad-action-scripts` port.

---

## At a glance

| API | Status in KiCad 10 | Fix |
|---|---|---|
| `VIATYPE_THROUGH` | Removed from bindings | Define manually as `3` |
| `Version() < "7"` pattern | Broken — lexicographic comparison | Use `_kicad_version_major() < 7` |
| `zone.HitTestInsideZone(pt)` | Removed | `zone.HitTest(pt)` |
| `zone.GetPriority()` | Renamed | `zone.GetAssignedPriority()` |
| `pcb.GetBoardPolygonOutlines(poly)` | New required arg | Add `True` as second arg |

---

## Details

### `VIATYPE_THROUGH`

- **What happened:** Removed from SWIG Python bindings in KiCad 10.
- **Symptom:** `NameError: name 'VIATYPE_THROUGH' is not defined`
- **Fix:**
```python
try:
    _ = VIATYPE_THROUGH
except NameError:
    VIATYPE_THROUGH = 3  # integer value of VIATYPE::THROUGH
```

---

### `Version()` string comparison

- **What happened:** `Version()` still works and returns `"10.0.0"`, but comparing it as a string is broken. `"10.0.0" < "7"` is `True` because `"1" < "7"` lexicographically.
- **Symptom:** KiCad 10 silently runs the KiCad < 7 code path everywhere.
- **Fix:**
```python
def _kicad_version_major():
    try:
        return int(Version().split('.')[0])
    except Exception:
        return 0

# Usage
if _kicad_version_major() < 7:
    ...
```

> [!warning] This is particularly nasty because it causes no errors — just wrong behaviour.

---

### `ZONE.HitTestInsideZone()`

- **What happened:** Method removed in KiCad 10.
- **Symptom:** `AttributeError` — or in the old code, silently swallowed by a bare `except`, disabling keepout zone enforcement.
- **Fix:** Check with `hasattr()` and fall back to `HitTest()`:
```python
if hasattr(zone, 'HitTestInsideZone'):
    result = zone.HitTestInsideZone(point)
else:
    result = zone.HitTest(point)  # KiCad 10+
```

---

### `ZONE.GetPriority()`

- **What happened:** Renamed to `GetAssignedPriority()` in KiCad 10.
- **Symptom:** `AttributeError: 'ZONE' object has no attribute 'GetPriority'`
- **Fix:**
```python
# Before
zone.GetPriority()

# After
zone.GetAssignedPriority()
```

---

### `BOARD.GetBoardPolygonOutlines(poly)`

- **What happened:** The `aInferOutlineIfNecessary` parameter became **required** in KiCad 10 (was optional with a default in KiCad 9).
- **Symptom:** `TypeError: GetBoardPolygonOutlines() missing 1 required positional argument`
- **Fix:** Try old signature first, fall back to new:
```python
try:
    pcb.GetBoardPolygonOutlines(board_edge)
except TypeError:
    pcb.GetBoardPolygonOutlines(board_edge, True)  # KiCad 10
```

---

## See also

- [[kicad10-what-changed]] — how these were applied in `kicad-action-scripts`
- [[kicad10-build-guide]] — rebuilding the PCM package
