# cloud-native-dsm

On-the-fly Digital Surface Model (DSM) of Pittsburgh, PA — no preprocessing.

Streams USGS 3DEP bare-earth DEM and lidar point cloud (LPC) directly from public
cloud storage, combines them into a surface model, and visualizes it interactively
in a [Marimo](https://marimo.io) notebook with [lonboard](https://github.com/developmentseed/lonboard)
/ deck.gl. NAIP imagery is draped over the surface for true-color context.

## Approach

- **Bare earth (DTM):** USGS 3DEP Seamless 1m DEM COGs from `s3://prd-tnm`, streamed via
  `obstore` + `async-geotiff`. (Proven — ported from `usgs-seamless-1m-viewer`.)
- **Surface (LPC):** USGS 3DEP lidar as COPC (Planetary Computer `3dep-lidar-copc`),
  streamed via `obstore` range-reads + `laspy`/`lazrs`. Classified into ground / vegetation
  / buildings.
- **Imagery:** NAIP true-color from Planetary Computer, draped over the surface.
- **Viz:** lonboard for point clouds; optional anywidget deck.gl `TerrainLayer` for a
  draped terrain mesh.

No `uv` venv — notebooks are self-contained via PEP 723 inline deps:

```bash
uv run marimo edit <notebook>.py --sandbox
```

## Scope

Start with a small AOI (a neighborhood). The cloud-native, no-preprocessing model is fine
for borough-sized areas; full regional coverage at 1m needs LOD (octree levels) and is
deferred. See [PLAN.md](PLAN.md).
