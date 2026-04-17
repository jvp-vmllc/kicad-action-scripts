---
title: kicad-action-scripts — KiCad 10 Port
tags:
  - kicad
  - plugin
  - via-stitching
  - kicad10
  - pcm
created: 2026-04-17
status: done
---

# kicad-action-scripts — KiCad 10 Port

> [!summary]
> This is the index note for the KiCad 10 compatibility port of [`kicad-action-scripts`](https://github.com/jsreynaud/kicad-action-scripts). The work is based on upstream [PR #94](https://github.com/jsreynaud/kicad-action-scripts/pull/94) and lives on the `pr94` branch of the fork.

---

## What is this?

`kicad-action-scripts` is a KiCad plugin that provides:
- **ViaStitching** — fills copper zones with stitching vias automatically
- **CircularZone** — creates circular keepout / copper zones

KiCad 10 introduced several breaking changes to its Python API that broke the plugin. This port fixes all of them.

---

## Notes in this set

| Note | Contents |
|---|---|
| [[kicad10-what-changed]] | Every code change made, file by file |
| [[kicad10-build-guide]] | How to build the `.zip` for KiCad's Plugin Manager |
| [[kicad10-api-reference]] | Quick reference card of all broken KiCad 10 APIs |

---

## Repository

| Item | Link |
|---|---|
| Fork | https://github.com/jvp-vmllc/kicad-action-scripts |
| Branch | `pr94` |
| Release | `v10.0-0.7.0` |
| Package | `kicad-action-scripts_v10.0-0.7.0.zip` |
| Upstream PR | https://github.com/jsreynaud/kicad-action-scripts/pull/94 |

---

## Install (quick)

1. Download `kicad-action-scripts_v10.0-0.7.0.zip` from the [release page](https://github.com/jvp-vmllc/kicad-action-scripts/releases/tag/v10.0-0.7.0)
2. KiCad → **Plugin and Content Manager** → **Install from File…**
3. Select the zip → restart KiCad if prompted

> [!tip] The plugin appears under **Tools → External Plugins → Via Stitching Generator**

---

## Related

- [[kicad10-what-changed]]
- [[kicad10-build-guide]]
- [[kicad10-api-reference]]
