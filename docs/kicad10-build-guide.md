---
title: KiCad 10 Port — Build Guide
tags:
  - kicad
  - kicad10
  - pcm
  - build
created: 2026-04-17
status: done
---

# KiCad 10 Port — Build Guide

> [!info] Parent note
> [[kicad-action-scripts-kicad10]]

How to rebuild the KiCad Plugin and Content Manager (PCM) `.zip` from source.

---

## Prerequisites

- **Python 3** (standard library only — no pip installs needed)
- The **repo** checked out on branch `pr94`
- The **previous release zip** as a donor for unchanged assets (CircularZone, icon):

```
https://github.com/jsreynaud/kicad-action-scripts/releases/download/v9.0-0.7.0/kicad-action-scripts_v9.0-0.7.0.zip
```

> [!note] Why borrow from the old zip?
> `CircularZone/` and `resources/icon.png` are not modified by this port and are not fully tracked in the repo (some binary assets). Pulling them from the last known-good release is the cleanest approach.

---

## Expected zip layout

KiCad PCM requires this exact directory structure inside the zip:

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

---

## Build script

Save as `build_pcm.py` anywhere (e.g. next to the repo root), edit the three path variables at the top, then run `python build_pcm.py`.

```python
import zipfile, os, json

# ── Edit these three paths ─────────────────────────────────────────────────
REPO    = r"C:\path\to\kicad-action-scripts"
REF_ZIP = r"C:\path\to\kicad-action-scripts_v9.0-0.7.0.zip"
OUT     = r"C:\path\to\kicad-action-scripts_v10.0-0.7.0.zip"
# ───────────────────────────────────────────────────────────────────────────

VIA_FILES = [
    "FillArea.py", "FillAreaAction.py", "FillAreaDialog.py",
    "__init__.py", "FillAreaTpl.fbp",
    "stitching-vias.png", "stitching-vias.svg",
    "stitching-vias-help.png", "stitching-vias-help.svg",
]
CIRCULAR_FILES = [
    "CircularZone.py", "__init__.py", "CircularZoneDlg.py",
    "CircularZoneDlg.fbp", "round_keepout_area.png", "round_keepout_area.svg",
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
        "Readme": "https://github.com/jsreynaud/kicad-action-scripts/blob/master/README.md",
    },
    "tags": ["annular", "pcbnew", "dxf", "grid", "mcad"],
    "versions": [{
        "version": "10.0.0",
        "status": "stable",
        "kicad_version": "10.0",
        "platforms": ["linux", "macos", "windows"],
    }],
}

with zipfile.ZipFile(OUT, "w", zipfile.ZIP_DEFLATED) as zout:
    # ViaStitching — from the repo (our modified files)
    for fname in VIA_FILES:
        src = os.path.join(REPO, "ViaStitching", fname)
        zout.write(src, f"plugins/ViaStitching/{fname}")
        print(f"  + plugins/ViaStitching/{fname}")

    # CircularZone + shared assets — unchanged, pulled from previous release zip
    with zipfile.ZipFile(REF_ZIP, "r") as zref:
        for fname in CIRCULAR_FILES:
            key = f"plugins/CircularZone/{fname}"
            zout.writestr(key, zref.read(key))
            print(f"  + {key}")
        zout.writestr("plugins/__init__.py", zref.read("plugins/__init__.py"))
        zout.writestr("resources/icon.png",  zref.read("resources/icon.png"))
        print("  + plugins/__init__.py")
        print("  + resources/icon.png")

    zout.writestr("metadata.json", json.dumps(METADATA, indent=2))
    print("  + metadata.json")

print(f"\nDone → {OUT}")
```

---

## How to install the zip in KiCad

1. Open **KiCad** (main window)
2. Click **Plugin and Content Manager**
3. Click **Install from File…**
4. Select `kicad-action-scripts_v10.0-0.7.0.zip`
5. Click **Apply Pending Changes**
6. Restart KiCad if prompted

> [!tip] After install the plugin appears under **Tools → External Plugins → Via Stitching Generator** inside `pcbnew`.

---

## Where to find the pre-built zip

The ready-to-install zip is attached to the GitHub release:

```
https://github.com/jvp-vmllc/kicad-action-scripts/releases/tag/v10.0-0.7.0
```

---

## See also

- [[kicad10-what-changed]] — every code change in detail
- [[kicad10-api-reference]] — API change quick-reference
