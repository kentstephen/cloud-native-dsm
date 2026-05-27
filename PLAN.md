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
**I/O decision: stay on `prd-tnm`, plain LAZ, no signed URLs, same `obstore` S3Store as the DEM.**
obstore does NOT need COPC — it does whole-object and range GETs on any bytes. COPC only unlocks
sub-tile range reads, which we don't need at small-AOI scale. So: obstore fetches whole LAZ tiles →
decode with `laspy`/`lazrs` → filter/color by classification. One cloud, one bucket, zero signing.

- LPC tile index (remote, vsicurl, do NOT download — 2.8 GB):
  `/vsicurl/https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/LPC/FullExtentSpatialMetadata/LPC_TESM.gpkg`
  Columns `tile_id, project, project_id, workunit_id, geometry` (CRS EPSG:4269 — bbox works directly).
- AOI coverage confirmed: **`PA_WesternPA_2019_D20`** (QL2 2019, 49 tiles for the full city bbox),
  fallback `PA_STATEWIDE_S_2006_2008_Legacy_Data`. Sample tile_id `17TNE589473`.
- TODO before coding: confirm the per-tile LAZ S3 path convention (index has `tile_id`/`project`
  but no file path). The project folder has metadata + the `LAZ/` subdir + a download manifest —
  browse it here to read off the tile→filename mapping for `17TNE589473`:
  http://prd-tnm.s3-website-us-west-2.amazonaws.com/?prefix=StagedProducts/Elevation/LPC/Projects/PA_WesternPA_2019_D20/

**Scale ceiling (decided):** whole-tile LAZ works for a neighborhood-to-city AOI (a few tiles, tens
of M points). It does NOT scale to "surrounding area" / county (~900 tiles, hundreds of GB, billions
of points — exceeds the lonboard/deck.gl render budget and breaks the no-preprocessing model).
Region-scale is deferred to Phase 6 via either (a) LOD over coarse octree levels — requires EPT/COPC,
the no-sign range-readable option is `s3://usgs-lidar-public` (still AWS, no signing); or
(b) rasterize Z→max to a 1m DSM grid and render that as terrain with overviews.
Planetary Computer COPC (Azure, SAS-signed) is last resort only.

### Phase 3 — Compose DSM
- Render classified LPC points over DEM terrain in lonboard. First real "DSM on the fly".

### Phase 4 — NAIP drape
- Reuse `naip_usgs_join_h3_1m.py` (drop H3). Drape for roads + true-color context.

### Phase 5 — Live controls
- Marimo reactive widgets: AOI bbox, class toggles, elevation exaggeration, point size, basemap.

### Phase 6 — Optional / later (region-scale + mesh)
- anywidget deck.gl `TerrainLayer` for draped mesh (vs. points).
- Region-scale LPC: LOD via EPT octree levels (`s3://usgs-lidar-public`, no-sign) OR rasterize
  Z→max to a 1m DSM grid rendered as terrain with overviews. (See Phase 2 scale ceiling.)

**Guardrail:** small AOI first; expand only after each phase verifies.

## Reference repos
- `usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py` — DEM COG streaming (port this).
- `usgs-seamless-1m-viewer/naip_usgs_join_h3_1m.py` — NAIP from Planetary Computer.
- `deckgl-raster-mapterhorn-s2/` — drape raster over terrain pattern.
