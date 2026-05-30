# CLAUDE.md — Agent context for Bathymetryst

This file orients coding agents working in this repo. Read it before making changes.
(Non-Claude tools: `AGENTS.md` points here.)

## What this project is

Bathymetryst visualizes how global coastlines shifted between the **last glacial
maximum** (low sea level, ~−120 m, ~20–26 ka) and the **present interglacial /
glacial minimum** (0 m). See `README.md` for the vision and the three goals.

## Architecture & data flow

```
fetch_dem.py      ─┐
fetch_basemap.py  ─┤→ data/raw/        (gitignored)
                   │
sea_level.py ──────┤  paleo curve → global sea-level offset per year-BP
                   ▼
model_shoreline.py →  data/processed/  land/water masks + area stats (gitignored)
                   ▼
build_manifest.py →  web/data/manifest.json   (committed; describes every slice)
                   ▼
web/ (MapLibre GL JS)  reads manifest.json + overlays → slider + stats
```

- **Pipeline** lives in `pipeline/` (Python). **Presentation** lives in `web/`
  (static, no build step — plain HTML/CSS/JS + MapLibre via CDN).
- The web app only ever reads `web/data/*` — it never touches the pipeline directly.
  The pipeline's contract with the web app is **`manifest.json`** (shape in
  `docs/data-model.md`).

## Modeling approach

- **MVP = bathtub model.** `model_shoreline.py`: `is_water = dem_elevation < sea_level_offset`.
  Compute land area per slice and `land_delta_km2` vs present.
- **Future = paleo-DEMs.** A `ShorelineLayer.source_method` of `"paleodem"` lets a
  more accurate (GIA-corrected) source replace bathtub output without schema changes.
- Drive everything with `sea_level.py`, which interpolates `data/sea_level_curve.csv`.

## Where things live

| Concern | Location |
|---------|----------|
| Tunable constants (DEM URLs, slice levels, bounds) | `pipeline/config.py` |
| Data model (dataclasses mirroring JSON Schemas) | `pipeline/models.py` |
| JSON Schemas (source of truth for shapes) | `data/schema/*.json` |
| Paleo sea-level control points | `data/sea_level_curve.csv` |
| Map init, basemap, city markers | `web/js/map.js` |
| Slider → active slice → overlay swap | `web/js/slider.js` |
| Land gained/lost readout | `web/js/stats.js` |

## Conventions & gotchas

- **CRS** is EPSG:4326 (lon/lat) everywhere unless a raster step states otherwise.
- **Sea-level sign:** offsets are **negative** below present datum. Glacial *maximum*
  = most negative offset = most land. Don't flip this.
- **Vertical datum:** DEM elevations are relative to present mean sea level; record the
  source datum in `DEMMetadata`. The bathtub model assumes a globally uniform offset.
- **Large files are gitignored** (`data/raw/`, `data/processed/`, generated tiles/PNGs).
  Only `manifest.json` and seed vectors are committed so the page loads.
- **No web build step.** Keep `web/` dependency-light; load MapLibre from CDN. Don't
  add bundlers/frameworks without good reason.
- **`sea_level.py` and `build_manifest.py` are stdlib-only** so they run without the
  heavy geo deps. `fetch_*`/`model_shoreline.py` need `requirements.txt`.

## Key commands

```bash
make setup     # venv + pip install -r requirements.txt
make data      # fetch → model → manifest
make manifest  # rebuild web/data/manifest.json from the sea-level curve
make serve     # python -m http.server in web/
python -m pipeline.sea_level         # sanity-check curve interpolation
python -c "import pipeline.models"   # data model imports cleanly
```

## Current state (scaffold)

- ✅ Data model, schemas, sea-level curve, manifest builder, web app shell.
- 🚧 `fetch_dem.py`, `fetch_basemap.py`, `model_shoreline.py` are **stubs with TODOs** —
  real DEM download/processing and overlay generation are the next work items.
- The web app runs against seed `manifest.json` + `cities.geojson` today.

## Working agreement

- Develop on `claude/coastline-glacial-timeline-2Pwd7`. Commit with clear messages
  and push so work is never lost.
- **Always open a pull request** for your changes. The PR description must state the
  **goals** of the change — what it accomplishes and why — alongside a summary of what
  changed. Keep the PR up to date as you push follow-up commits.
- When you change a data shape, update **both** `data/schema/*.json` and
  `pipeline/models.py`, and note it in `docs/data-model.md`.
