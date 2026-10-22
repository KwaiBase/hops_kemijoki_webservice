# HOPS Hydrological Prediction, Comparison & Visualisation System

Modernized mock implementation of the HOPS hydrological forecast visualization and comparison service.

## What is included

- React frontend served from `frontend/`
- Leaflet dual-map main view with synchronized pan/zoom
- Basin detail view with map, linked time controls, dual-variable plot, streamflow plot, and RMSE/NSE/KGE statistics
- Config-driven variables, models, basins, logos, and CMS pages
- File-backed data API with operational CSV/PNG data supplied outside the repository
- Markdown CMS pages under `content/pages/`
- Admin/config page for editing guidance and live configuration inspection
- Python server with FastAPI-compatible API shape and no mandatory third-party dependency

## Run locally

```powershell
python .\server.py
```

Set `HOPS_DATA_DIR` to a data root containing the operational basin, observation,
PNG, and GeoJSON data before starting the service. The repository does not store
operational CSV or PNG files; use a small local fixture directory for development
or the mounted data directory used by OpenShift.

Open:

```text
http://127.0.0.1:8000
```

If you install FastAPI later, the same folder can be migrated to `uvicorn` without changing frontend API calls.

Health check:

```text
http://127.0.0.1:8000/health
```

## Environment variables

The app does not require hardcoded local paths. These variables are read at runtime:

```text
HOPS_HOST=127.0.0.1
HOPS_PORT=8000
HOPS_BASE_PATH=
HOPS_FRONTEND_DIR=frontend
HOPS_CONFIG_FILE=config/app.json
HOPS_CONTENT_DIR=content
HOPS_DATA_DIR=data
HOPS_DATA_BASE_URL=/data
HOPS_TILE_URL=https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png
```

`HOPS_BASE_PATH` supports reverse-proxy or OpenShift Route prefixes, for example `/hops`.

`HOPS_DATA_DIR` can point to a mounted persistent volume. `HOPS_DATA_BASE_URL` can later point to a public object storage prefix or CDN, while API basin timeseries can continue to read from the mounted data directory.

## Run with a container

Build:

```powershell
podman build -t hops-v2 -f .\Containerfile .
```

Run:

```powershell
podman run --rm -p 8000:8000 --env HOPS_HOST=0.0.0.0 hops-v2
```

Run with externally supplied data:

```powershell
podman run --rm -p 8000:8000 --env HOPS_HOST=0.0.0.0 --volume C:\path\to\hops-data:/mnt/hops-data:ro --env HOPS_DATA_DIR=/mnt/hops-data hops-v2
```

## OpenShift notes

- Environment-specific OpenShift manifests are kept locally and are not committed to this public repository. See `Deploy_on_OpenShift.md` for the deployment workflow.
- Build from `Containerfile` using OpenShift Builds, Shipwright, Tekton, or an external registry.
- Expose container port `8000`.
- Configure readiness/liveness probes against `/health`.
- Set `HOPS_HOST=0.0.0.0`.
- Set `HOPS_BASE_PATH` only if the application is served below a subpath instead of route root.
- Mount operational PNG/CSV/GeoJSON data as a PVC at a path such as `/mnt/hops-data` and set `HOPS_DATA_DIR=/mnt/hops-data`.
- If static PNG/GeoJSON layers are moved to object storage, set `HOPS_DATA_BASE_URL` to that object storage or CDN URL.
- Keep `config/app.json` and `content/pages/*.md` in the image for simple deployments, or mount them from ConfigMaps for operational editing.
- The frontend uses same-origin API calls and runtime configuration from `/runtime-config.js`, so route hostnames do not need to be compiled into the frontend.

## Data layout

```text
data/
  png/hops/{variable}/{date}.png       # supplied externally
  png/ecmwf/{variable}/{date}.png
  png/hsaf/{variable}/{date}.png
  png/clms/{variable}/{date}.png
  basins/hops/{basin}.csv              # supplied externally
  basins/ecmwf/{basin}.csv
  basins/hsaf/{basin}.csv
  basins/clms/{basin}.csv
  basins/streamflow/{basin}_obs.csv
  basins/streamflow/{basin}_{model}.csv
  metobs/fmi_tempc_{station_id}.csv    # supplied externally
  metobs/fmi_precip_{station_id}.csv
  watersheds/{basin}.geojson
  geojson/rivers.geojson
config/
  app.json
  observation-stations.json             # supplied externally for station locations
  map-options.json                      # supplied externally for map/plot selections
  display-options.json                  # supplied externally for display defaults
content/
  pages/*.md
```

The tracked `data/` directory contains only small geographic assets such as
`geojson/` and `watersheds/`. Operational `basins/`, `metobs/`, and `png/`
directories are ignored by Git and must be copied or mounted separately when
running data-dependent views locally.

## Extending

- **Define the variable.** Add one object to the `variables` array in `config/app.json`. Give it a unique `id`, plus `label`, `unit`, `palette`, `source`, and `group`. For example:

  ```json
  {"id": "open_example", "label": "Open Example", "unit": "mm", "palette": "#55c7e8", "source": "hops", "group": "Open"}
  ```

  `id` is the common key used in data filenames/columns and the other configuration files. Use one of the supported source IDs: `hops`, `ecmwf`, `hsaf`, or `clms`. The `group` is a UI selector heading; it does not create a data directory. For a map-only variable, add `"basinTimeseries": false`.

- **Allow it in the relevant selectors.** Add the ID to `mapVariables` in `config/map-options.json` for map selection (main maps and the basin detail map). Add it to `basinVariables` in that file if it has basin-average timeseries data and should be selectable in basin plots. These are arrays of IDs, not full variable objects. A basin-timeseries variable should not set `basinTimeseries` to `false`.

- **Place its data under `HOPS_DATA_DIR`.** The tracked repository does not contain operational PNG/CSV data; in OpenShift the data root is `/mnt/hops/data`. For each supported source, paths are:

  | `source` in `app.json` | Raster, one PNG per date | Basin averages, one CSV per basin |
  | --- | --- | --- |
  | `hops` | `png/hops/{id}/{yyyy-mm-dd}.png` | `basins/hops/{basin}.csv` |
  | `ecmwf` | `png/ecmwf/{id}/{yyyy-mm-dd}.png` | `basins/ecmwf/{basin}.csv` |
  | `hsaf` | `png/hsaf/{id}/{yyyy-mm-dd}.png` | `basins/hsaf/{basin}.csv` |
  | `clms` | `png/clms/{id}/{yyyy-mm-dd}.png` | `basins/clms/{basin}.csv` |

  For example, the HOPS raster for `open_example` on 2026-10-22 is `/mnt/hops/data/png/hops/open_example/2026-10-22.png`; its basin value column goes in `/mnt/hops/data/basins/hops/6501700.csv`. Basin CSVs need a `date` column and a column named exactly after each variable ID. Put each variable's data in the CSV for its source. The API merges the four supported source files by date. Values at or below `noDataThreshold` (normally `-99998`) are treated as missing.

- **Configure display behavior if needed.** Edit `config/display-options.json` (preserve the current settings in that file). No entry is required for a new variable unless you need an override: `variableColors` maps IDs to palette overrides, `legends` maps IDs to map legend definitions, and `enabledVariables` maps IDs to visibility flags. `enabledGroups` maps exact `group` labels to flags; omitted flags default to enabled. Add an ID to `map.mainDefaultVariable`, `map.comparisonDefaultVariable`, `map.basinDefaultVariable`, or the `plotDefaults` variable fields only when it should be selected by default. If values can be negative, add its ID to `plotScaling.temperatureVariables` so the basin plot uses a signed axis.

- **Add a category/group within an existing source.** Set the new exact category label as `group` in each new variable definition. The dropdown heading is generated from that field, so no category-specific data folder is needed: use the same source/ID paths above. Add the variables to `mapVariables` and/or `basinVariables` as appropriate. Add the category label to `enabledGroups` in `display-options.json` if operators should be able to enable/disable the whole group. If this is a land-cover category requiring a mask, also add its GeoJSON mask under `data/geojson/` and extend `VARIABLE_MASKS`/`maskForVariable` in `frontend/app.js`; the existing `open`, `forest`, and `bog` prefixes are the only ones recognized automatically.

- **Adding a new source requires code changes.** The current frontend source list and backend basin-file merge support only HOPS, ECMWF, HSAF, and CLMS. Extend `SOURCE_OPTIONS` in `frontend/app.js` and `read_basin_rows` in `server.py` before adding a source to variable definitions or expecting its map/basin data to load.

- **Deploy the changes.** Locally, the bundled config files are used by default. In OpenShift, `map-options.json` and `display-options.json` are read from `/mnt/hops/config/` at runtime, so their changes apply on refresh without rebuilding. `app.json` has no mounted override and is baked into the image: changing the variable catalog requires building and rolling out a new image. Keep operational PNG/CSV data on the mounted data volume, not in Git.

- Add streamflow observation data in `data/basins/streamflow/{basin}_obs.csv`; use `flag` value `1` for unreliable observations
- Add streamflow model data in `data/basins/streamflow/{basin}_{model}.csv`
- Add basins in `config/app.json`; add basin GeoJSON and timeseries file using the basin id
- Set the Obs/Sim label position per station with the optional `"labelPosition"` field (`"top"`, `"right"`, `"bottom"` or `"left"`, default `"right"`) in `observation-stations.json`
- Add meteorological observation stations in `config/observation-stations.json`; add station CSV files in `data/metobs/fmi_tempc_{station_id}.csv` or `data/metobs/fmi_precip_{station_id}.csv`
- Set every default map and plot from `display-options.json` (sources are picked automatically from the variable ID; `"none"` is allowed for plot variables). Maps, under `map`: `mainDefaultVariable`, `comparisonDefaultVariable`, `basinDefaultVariable`, `showRivers`, `showLegendA`/`showLegendB`, `maskA`/`maskB`, `basinMapMask`, `basinShowLegend`, `river` (`color`, `weight`, `opacity` of the river lines). Basin plots, under `plotDefaults`: `topMode`/`bottomMode` (`basin`, `streamflow`, `none`), `topVariableA/B`, `topKindA/B`, `bottomVariableA/B`, `bottomKindA/B` (`line` or `bar`), `topModels`/`bottomModels`, `topShowObservations`/`bottomShowObservations`, `topShowStats`/`bottomShowStats`, `days` (one of `timeControls.ranges`), `topPlotPercent`. Y-axis scaling is configured under `plotScaling` (see "Plot y-axis scaling" below)
- Set the legend box width in pixels with `legendWidth` in `display-options.json` (default 150); a single legend can override it with `"width"` inside its `legends` entry
- Configure map legends in the `legends` object of `config/display-options.json`; each map has an independent Legend visibility toggle
- Edit CMS pages and place figures under `/mnt/hops/content/pages/` in OpenShift; the app falls back to the bundled `content/pages/` files when an external page is absent
- Edit CMS text in `content/pages/*.md`

## Plot y-axis scaling

Basin view plots (basin averages and streamflow) share one scaling scheme, implemented in the `Plot` component in `frontend/app.js` and configured under `plotScaling` in `display-options.json` (mounted in OpenShift, so changes need no rebuild):

```json
"plotScaling": {
  "headroom": 0.1,
  "temperatureHeadspace": 0.5,
  "temperatureToespace": 1.0,
  "temperatureVariables": ["hres_2m_air_temp"]
}
```

- **Yearly scale.** The axis range comes from all rows of the *selected year* (including forecast rows), not from the visible period. The scale therefore stays the same when switching 1M/3M/6M/1Y or drag-zooming, and periods stay comparable.
- **Normal variables** (streamflow, runoff, precipitation, SWE, volumes, ...): the axis runs from 0 to the year's maximum x (1 + `headroom`). Each axis (left/right) is scaled separately from the series on it.
- **Signed variables** (air temperature): the axis runs from the year's minimum minus `temperatureToespace` to the year's maximum plus `temperatureHeadspace`. Both are in data units (degrees), not percentages. `temperatureToespace` (space below the yearly minimum) falls back to `temperatureHeadspace` if not set.
  - A faint dashed zero line is drawn on the axis when 0 is inside the range.
  - Bars start at the zero line and go up or down from it, instead of starting at the bottom of the plot.
- **A variable counts as signed only if its ID is in `temperatureVariables`** (default `["hres_2m_air_temp"]`). When adding a new variable that can be negative (for example another temperature, or an anomaly), add its ID to that list. Otherwise it is scaled from 0 and negative values are cut off.
- Normal axes never go below 0 unless the data does; a series with no positive values gets an axis of 0 to 1.
- If a plot has the signed variable on one axis and a normal variable on the other, each axis uses its own rule. The zero line is drawn for the signed axis only.
- Missing or invalid `plotScaling` values fall back to the defaults above. The older `plotDefaults.streamflowHeadroom` key is still read as a fallback for `headroom`.

When adding new data: new basin-average columns are scaled automatically with the yearly rule. Only variables that need a signed axis have to be listed in `temperatureVariables`.
