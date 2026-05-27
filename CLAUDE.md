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
| Lidar (LPC, surface) | Planetary Computer `3dep-lidar-copc` | public STAC | `obstore` range-reads + `laspy`/`lazrs` |
| Imagery | NAIP (Planetary Computer) | public STAC | `rioxarray` |

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

## Guardrail

Start with a small AOI (a neighborhood). The no-preprocessing, no-cache model is fine for
borough-sized areas; full-region 1m will choke the browser. Wide-area is deferred to LOD via the
COPC octree levels.

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
