# Plan — cloud-native-dsm

## Goal
Build an on-the-fly DSM of Pittsburgh, PA in a Marimo notebook: combine streamed USGS
bare-earth DEM + lidar point cloud (LPC) into a surface model, drape NAIP, no preprocessing.

## Feasibility verdict: GO
- DEM streaming: proven (`usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py`, Pittsburgh bbox already set).
- LPC: USGS 3DEP lidar is cloud-native via COPC (Planetary Computer) — range-readable, fits obstore.
- Pittsburgh / Allegheny County has full QL2 lidar coverage.
- Caveat: keep AOI small initially. Full region at 1m, no-cache, will choke the browser.

## Data sources
| Layer | Source | Access | Reader |
|---|---|---|---|
| Bare earth DEM | `s3://prd-tnm` 3DEP Seamless 1m COGs | public, no-sign | `obstore` + `async-geotiff` |
| LPC (surface) | Planetary Computer `3dep-lidar-copc` | public STAC | `obstore` range-reads + `laspy`/`lazrs` |
| Imagery | NAIP (Planetary Computer) | public STAC | `rioxarray` (reuse existing code) |

Lidar classification codes: ground=2, low/med/high veg=3/4/5, buildings=6.
Roads are not a lidar class — they come from the NAIP drape. Bridges (class 17) are
inconsistently populated; don't rely on them.

## Phases

### Phase 0 — Repo setup  ✅ (in progress)
- git init, name `cloud-native-dsm`, `.gitignore`, README. Done.
- No uv venv; PEP 723 inline deps + `marimo edit --sandbox`.

### Phase 1 — DEM terrain (port proven code)
- Lift DEM-streaming cells into a fresh notebook. Render ground in lonboard. Verify end-to-end first.

### Phase 2 — LPC streaming (new work)
- COPC from Planetary Computer; confirm Pittsburgh tile coverage.
- obstore range-reads → `laspy`/`lazrs` decode. Filter/color by classification.

### Phase 3 — Compose DSM
- Render classified LPC points over DEM terrain in lonboard. First real "DSM on the fly".

### Phase 4 — NAIP drape
- Reuse `naip_usgs_join_h3_1m.py` (drop H3). Drape for roads + true-color context.

### Phase 5 — Live controls
- Marimo reactive widgets: AOI bbox, class toggles, elevation exaggeration, point size, basemap.

### Phase 6 — Optional / later
- anywidget deck.gl `TerrainLayer` for draped mesh (vs. points).
- Gridded DSM rasterization; LOD via COPC/EPT octree levels for wider area.

**Guardrail:** small AOI first; expand only after each phase verifies.

## Reference repos
- `usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py` — DEM COG streaming (port this).
- `usgs-seamless-1m-viewer/naip_usgs_join_h3_1m.py` — NAIP from Planetary Computer.
- `deckgl-raster-mapterhorn-s2/` — drape raster over terrain pattern.
