# Maine Lakes Monitoring — Architecture Notes

Three decisions shaped the platform design.

---

## 1. Heterogeneous data integration behind a single API

The platform draws from five fundamentally different data sources: historical field measurements (decades of manual monitoring records), satellite-derived indicators (Landsat, Sentinel-2, Sentinel-3), hydrodynamic model outputs (GLM 1-D), meteorological forecasts (NOAA GFS), and geospatial lake geometry (HydroLAKES-derived polygons). Each source has different formats, update cadences, resolution, and coordinate systems.

Rather than exposing this complexity to the frontend, the Python/FastAPI backend normalizes all sources into a consistent query interface. The Angular frontend makes simple requests — "give me chlorophyll-a for Lake Auburn between 2015 and 2020" — and the backend handles source routing, unit normalization, and result merging. This separation kept the frontend maintainable and made it possible to add new data sources without frontend changes.

---

## 2. Separate processing pipeline from serving layer

With 13M+ satellite observations and decades of field records, data preparation cannot happen at query time. A separate Python pipeline ingests, processes, resamples, and stores pre-aggregated values in MongoDB before they ever reach the API.

The serving layer (FastAPI) is read-only against processed data — it does not run scientific computations or transformations at request time. This keeps API response times predictable regardless of dataset size, and means heavy processing work (geospatial resampling, satellite band math, model output parsing) runs offline where it can be monitored, retried, and version-controlled independently.

---

## 3. Spatial queries via polygon-first design

The interactive map covers 2,807 lake polygons. A naive approach — loading all polygons at once or running bounding-box queries — would produce slow initial loads and poor zoom-level performance.

The approach: lake polygons are stored with spatial indexing and served at appropriate detail levels for the current map zoom. The frontend uses MapLibre GL's vector tile rendering for the base map layer, with Deck.gl overlays for satellite and model heatmap layers that need dynamic data binding. This keeps the geospatial interface responsive even when users pan across the full state or switch between spatial data layers.
