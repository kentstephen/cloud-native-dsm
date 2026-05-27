# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**cloud-native-dsm** — an interactive, on-the-fly Digital Surface Model (DSM) of Pittsburgh, PA,
built in a Marimo notebook. Streams USGS bare-earth DEM + lidar point cloud (LPC) directly from
public cloud storage, combines them into a surface model, and drapes NAIP imagery — **no
preprocessing**, analytics applied live. Intended to be share-worthy (LinkedIn/GitHub).

Status: early. See `PLAN.md` for the phased course of action; only repo scaffolding exists so far.

## Environment

No committed venv. Notebooks are self-contained via PEP 723 inline script metadata and run
sandboxed:

```bash
uv run marimo edit dsm.py --sandbox
```

`uv` installs the inline deps into an ephemeral environment. Add new deps to the notebook's
`# /// script` header, not a `pyproject.toml`.

## Architecture (target)

Three layers, all streamed remotely, composed in lonboard / deck.gl inside Marimo:

| Layer | Source | Access | Reader |
|---|---|---|---|
| Bare earth DEM (DTM) | `s3://prd-tnm` 3DEP Seamless 1m COGs | public, no-sign | `obstore` `S3Store` + `async-geotiff` |
| Lidar (LPC, surface) | `s3://prd-tnm` 3DEP LPC plain LAZ | public, no-sign | `obstore` whole-tile GET + `laspy`/`lazrs` |
| Imagery | NAIP (Planetary Computer) | public STAC | `rioxarray` |

**I/O decision: one bucket (`prd-tnm`), no signed URLs, plain LAZ.** obstore does NOT need COPC — it
does whole-object and range GETs on any bytes; COPC only unlocks sub-tile range reads we don't need
at small-AOI scale. So LPC = obstore fetches whole LAZ tiles → `laspy`/`lazrs` decode → filter/color
by class. Same `S3Store(skip_signature=True)` as the DEM. PDAL is not used (it owns its own I/O and
can't sit behind obstore); keep it in reserve only if we later want its `writers.gdal` DSM rasterizer.

### LPC tile index (analog of the S1M GeoPackage)

Find which LPC tiles overlap an AOI via the LPC spatial-extent index — read it **remotely** with
DuckDB `ST_Read` over `/vsicurl/` (it's 2.8 GB; do NOT download it):

```
/vsicurl/https://prd-tnm.s3.amazonaws.com/StagedProducts/Elevation/LPC/FullExtentSpatialMetadata/LPC_TESM.gpkg
```

- Columns: `fid, tile_id, project, project_id, workunit_id, geometry`. Geometry CRS is
  **EPSG:4269** (NAD83 lat/lon) — our WGS84 bbox works directly, no transform needed.
- For the Pittsburgh AOI (`-80.048, 40.407 → -79.935, 40.487`), the modern coverage is
  **`PA_WesternPA_2019_D20`** (QL2, 2019; 49 tiles). Older fallback:
  `PA_STATEWIDE_S_2006_2008_Legacy_Data`.
- Sample `tile_id`: `17TNE589473` (UTM zone 17T MGRS-style). Full per-tile LAZ S3 path under
  `s3://prd-tnm/StagedProducts/Elevation/LPC/Projects/<project>/...` — **path convention not yet
  confirmed**; resolve by listing the project prefix before coding Phase 2.

Note: Planetary Computer COPC assets are on **Azure blob**, not S3 — reached via STAC + signed
URLs, not an `s3://` URI. The native S3 LPC is the `prd-tnm` LAZ above (and the EPT mirror at
`s3://usgs-lidar-public`).

Key points future sessions need:

- **DEM streaming is proven** — port it from `usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py`
  (DuckDB spatial query on the S1M tile-index GeoPackage → intersecting tiles → `async-geotiff`
  windowed reads of a chosen overview → reproject EPSG:6350→4326 → lonboard `PointCloudLayer`).
  That file already has a Pittsburgh bbox. Despite its `lpc` filename it renders DEM raster
  pixels, **not** lidar.
- **COPC is the LPC path because it's range-readable**, which fits `obstore`. Fetch header +
  octree byte ranges with `obstore`, decode chunks with `laspy`/`lazrs`. This keeps a single
  obstore-based streaming model across DEM and LPC. (EPT via PDAL `readers.ept` is the
  alternative but does not use obstore.)
- **Lidar classification codes:** ground=2, low/med/high veg=3/4/5, buildings=6. Filter/color by
  class to get structures + trees above bare earth = the DSM.
- **Roads are not a lidar class** — they come from the NAIP drape. Bridges (class 17) are
  inconsistently populated; do not rely on them.

## Guardrail / scale ceiling

Start with a small AOI (a neighborhood). Whole-tile LAZ via obstore is fine for a neighborhood-to-
city AOI (a few tiles, tens of M points). It does NOT scale to "surrounding area" / county (~900
tiles, hundreds of GB, billions of points — exceeds the lonboard/deck.gl render budget and breaks
no-preprocessing). Region-scale is deferred to: (a) LOD over EPT octree levels at
`s3://usgs-lidar-public` (still AWS, no signing), or (b) rasterize Z→max to a 1m DSM grid rendered as
terrain with overviews. Planetary Computer COPC (Azure, SAS-signed) is last resort only.

## Reference repos (reuse, don't reinvent)

- `usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py` — DEM COG streaming (port directly).
- `usgs-seamless-1m-viewer/naip_usgs_join_h3_1m.py` — NAIP from Planetary Computer (drop the H3 join).
- `usgs-seamless-1m-viewer/CLAUDE.md` — DuckDB spatial + lonboard patterns and known issues
  (CRS warning, `always_xy :=` named param, GeoArrow auto-export, seamless-3dep timeouts).
- `deckgl-raster-mapterhorn-s2/` — drape-raster-over-terrain pattern (optional terrain mesh).

## Tone & Conduct

- No praise, flattery, or hedging. Respond directly; treat the user as a peer.
- Don't explain things not asked for. No unsolicited value judgments.
- If something is wrong, say so plainly — don't sandwich or soften. Never say "you're absolutely right".
- Memories go in `.claude/memory/` in this repo (gitignored), never the global auto-memory path.
