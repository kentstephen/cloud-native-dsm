# New plan - DSM of pittsburg, PA mairmo noteobook with deck.gl terrain layer

* I have code that allowed me to stream DEM seamless 1 meter from USGS s3 directly, i believe the same can be found for LPC, the python code can be found in this directory /Users/stephenk/dev/projects/usgs-seamless-1m-viewer/dem_s1m_lpc_viewer.py that same directory holds code for using NAIP
* the goal is to combine bare earth dem with lpc, querying both remotely initially using obstore then visaulizing the dem with lpc to create a dsm on the fly no preprocessing besides applying the analytics. specifically interested in Pittsburg PA and surrounding area.
    * I want structures and roads and bridges as well as dem, maybe even trees
    * visualizing LPC is difficult but maybe we will need to do that in order to get this right. lonboard has a way
* we can vis the dsm on its own, but i was able to drape raster over terrain in this repo /Users/stephenk/dev/projects/deckgl-raster-mapterhorn-s2/ although we are going to use deck terrain inside marimo, i believe this is possible. python functions can return html. also we can use the method of getting the naip data from microsoft: /Users/stephenk/dev/projects/usgs-seamless-1m-viewer/naip_usgs_join_h3_1m.py (we're not using h3)

## Is this project worth pursuing?
* we should determine that before beginning
* i would like to use mairmo bc its better set up for adjusting parameters to the map with a live reload
* i'm not sure we need a uv venv right away, we can just load deps on the fly or in sandbox mode, frees up space on my computer but start and create a git repository
* i have a lot of codebase, do the internet reseach you need to if the cloudnative approach will work
