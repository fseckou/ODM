# Requirements: Farmland Data Map Display via NodeODM

## Context & Vision

Build a custom web application that uses **NodeODM** as the processing backend and displays generated geospatial artifacts (orthophotos, DEMs, vegetation indices, contours) on our own interactive map — similar to what WebODM does, but tailored for **agent-driven farmland analysis**. AI agents will orchestrate drone image processing, interpret results, and provide actionable farming insights.

---

## 1. How WebODM Works Today (Analysis)

### 1.1 Architecture Overview

WebODM is a Django + Django REST Framework application that:
- Submits processing jobs to **NodeODM** (a Node.js REST API wrapper around the ODM CLI)
- Stores project/task metadata in PostgreSQL
- Serves map tiles via a **built-in Python tiler** (`app/api/tiler.py`) using `rio-tiler` / `cogeo`
- Renders maps client-side with **Leaflet.js**
- Extends functionality through a **plugin system** (Plant Health, Contours, etc.)

**Full WebODM stack:**
```
Browser (React + Leaflet + Potree for 3D point clouds)
    ↓
Nginx (reverse proxy)
    ↓
Gunicorn (WSGI)
    ↓
Django + Django REST Framework
    ├── app/api/tiler.py     → tile serving (rio-tiler 2.1.2 + rasterio 1.3.10)
    ├── app/api/formulas.py  → 24 vegetation index formulas
    ├── app/api/urls.py      → REST API routing
    ├── Celery workers       → background tasks (image resize, result retrieval)
    └── NodeODM connection   → processing engine (port 3000)
```

**Key insight:** WebODM does NOT use TiTiler or a separate tile server. Django serves tiles directly via rio-tiler's `COGReader` in the same process. There is no tile pre-rendering — tiles are generated on-the-fly from COG/GeoTIFF files on each request.

### 1.2 Tile Serving

WebODM serves raster tiles **on-the-fly from COG (Cloud Optimized GeoTIFF)** files:

| Endpoint Pattern | Purpose |
|---|---|
| `/api/projects/{pid}/tasks/{tid}/{type}/tiles.json` | TileJSON metadata |
| `/api/projects/{pid}/tasks/{tid}/{type}/bounds` | Geographic bounds |
| `/api/projects/{pid}/tasks/{tid}/{type}/metadata` | Raster metadata (bands, stats, min/max) |
| `/api/projects/{pid}/tasks/{tid}/{type}/tiles/{z}/{x}/{y}.png` | XYZ map tiles |
| `/api/projects/{pid}/tasks/{tid}/{asset}/export` | Full raster download |

Where `{type}` is one of: `orthophoto`, `dsm`, `dtm`.

The tiler reads COG files directly using `rio-tiler`, extracting only the bytes needed for the requested tile (no pre-tiling required). This is the key architectural insight — **COGs enable on-demand tile serving without a tile cache**.

### 1.3 Vegetation Index (VI) Generation

WebODM computes vegetation indices **entirely server-side** in `app/api/tiler.py`:

**How it works:** The frontend (React + Leaflet in `Map.jsx`) detects available bands from the orthophoto metadata, selects the appropriate formula name, and passes it as a query parameter on tile requests (`?formula=NDVI&bands=auto&color_map=rdylgn`). The server applies the formula via `lookup_formula()` from `app/api/formulas.py`, applies a colormap, and returns pre-rendered colored tiles. The browser never does band math — it only receives already-colored PNG tiles.

**Auto-selection logic:**
- Multispectral with NIR band → requests `NDVI`
- RGB-only → requests `VARI`
- Thermal (2-band LWIR) → requests `Celsius` with `color_map=magma`

**All 24 formulas** (defined in `app/api/formulas.py`):

| Index | Formula | Bands | Farmland Use |
|---|---|---|---|
| **NDVI** | `(N - R) / (N + R)` | NIR, R | Crop vigor, biomass estimation |
| **NDRE** | `(N - Re) / (N + Re)` | NIR, RE | Chlorophyll / nitrogen status |
| **GNDVI** | `(N - G) / (N + G)` | NIR, G | Canopy nitrogen |
| **NDWI** | `(G - N) / (G + N)` | G, NIR | Water stress detection |
| **NDYI** | `(G - B) / (G + B)` | G, B | Flowering detection (canola) |
| **ENDVI** | `((N+G) - 2B) / ((N+G) + 2B)` | NIR, G, B | Enhanced vegetation |
| **EVI** | `2.5*(N-R) / (N+6R-7.5B+1)` | NIR, R, B | High-biomass areas |
| **LAI** | `3.618*EVI - 0.118` | NIR, R, B | Leaf area index |
| **SAVI** | `1.5*(N-R) / (N+R+0.5)` | NIR, R | Sparse vegetation / bare soil |
| **OSAVI** | `(N-R) / (N+R+0.16)` | NIR, R | Variable soil backgrounds |
| **VARI** | `(G - R) / (G + R - B)` | RGB | Visible-light vegetation (no NIR) |
| **EXG** | `2*G - R - B` | RGB | Excess green (weed detection) |
| **MPRI** | `(G - R) / (G + R)` | G, R | Modified photochemical reflectance |
| **GLI** | `(2G - R - B) / (2G + R + B)` | RGB | Green leaf index |
| **vNDVI** | `0.5268*(R^-0.1294 * G^0.3389 * B^-0.3118)` | RGB | Visible-band NDVI approximation |
| **BAI** | `1 / ((0.1-R)^2 + (0.06-N)^2)` | R, NIR | Burn area index |
| **GRVI** | `N / G` | NIR, G | Green ratio |
| **MNLI** | `(N^2 - R)*1.5 / (N^2 + R + 0.5)` | NIR, R | Modified non-linear index |
| **MSR** | `((N/R)-1) / (sqrt(N/R)+1)` | NIR, R | Modified simple ratio |
| **RDVI** | `(N-R) / sqrt(N+R)` | NIR, R | Renormalized difference |
| **TDVI** | `1.5*(N-R) / sqrt(N^2 + R + 0.5)` | NIR, R | Transformed difference |
| **ARVI** | `(N - 2R + B) / (N + 2R + B)` | NIR, R, B | Atmospherically resistant |
| **Celsius** | `L` | LWIR | Thermal / irrigation monitoring |
| **Kelvin** | `L*100 + 27315` | LWIR | Thermal (raw) |

Results are false-colored using configurable color ramps (default: `rdylgn` diverging palette). Users can adjust rescaling (min/max) interactively via query parameters.

### 1.4 Contour Line Generation

WebODM generates contours **on-demand** (not pre-computed) via its Contours core plugin (`coreplugins/contours/api.py`):

1. User requests contours with a chosen interval (e.g., 1m, 0.25m)
2. Server runs a **pure GDAL pipeline** (current versions replaced GRASS GIS):
   - `gdalwarp` — crops DEM to task boundary
   - `gdal_contour` — extracts contour isolines with elevation attributes
   - `ogr2ogr` — reprojects/converts to requested output format
3. For preview: GeoJSON returned to Leaflet as vector overlay
4. For export: served via `TaskContoursDownload` endpoint

| Parameter | Default | Purpose |
|---|---|---|
| `layer` | required | DSM or DTM source |
| `epsg` | 3857 | Output coordinate reference system |
| `interval` | 1 | Contour spacing in meters |
| `format` | GPKG | Output format (GPKG, Shapefile, DXF, GeoJSON) |
| `simplify` | 0.01 | Line simplification tolerance |
| `zfactor` | 1 | Vertical exaggeration factor |

### 1.5 ODM Output Artifacts

ODM generates these artifacts (with relevant flags):

```
project/
  odm_orthophoto/
    odm_orthophoto.tif          # Main orthophoto (GeoTIFF or COG with --cog)
  odm_dem/
    dsm.tif                     # Digital Surface Model (--dsm)
    dtm.tif                     # Digital Terrain Model (--dtm)
  odm_georeferencing/
    odm_georeferenced_model.laz # Point cloud
    *.bounds.geojson            # Crop boundary
  odm_texturing/
    odm_textured_model_geo.obj  # 3D textured mesh
    odm_textured_model_geo.glb  # glTF binary (--gltf)
  odm_report/
    shots.geojson               # Camera positions
    stats.json                  # Processing statistics
  orthophoto_tiles/             # Pre-rendered XYZ tiles (--tiles)
  dsm_tiles/                    # Colored hillshade tiles (--tiles --dsm)
  dtm_tiles/                    # Colored hillshade tiles (--tiles --dtm)
  3d_tiles/                     # OGC 3D Tiles (--3d-tiles)
  entwine_pointcloud/           # EPT indexed point cloud (--pc-ept)
```

### 1.6 NodeODM API

NodeODM exposes a REST API (default port 3000):

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/info` | Server info, version, available options |
| `POST` | `/task/new` | Create processing task (small uploads, multipart) |
| `POST` | `/task/new/init` | Initialize chunked upload (large datasets) |
| `POST` | `/task/new/upload/{uuid}` | Upload chunk for initialized task |
| `POST` | `/task/new/commit/{uuid}` | Commit and start processing |
| `GET` | `/task/{uuid}/info` | Task status, progress, processing time |
| `GET` | `/task/{uuid}/output` | Processing console output (streaming) |
| `GET` | `/task/{uuid}/download/all.zip` | Download all results as zip archive |
| `POST` | `/task/{uuid}/cancel` | Cancel running task |
| `POST` | `/task/{uuid}/remove` | Remove task and results |
| `POST` | `/task/{uuid}/restart` | Restart a failed/cancelled task |
| `GET` | `/task/list` | List all tasks |

Key processing options for farmland use: `--dsm`, `--dtm`, `--cog`, `--orthophoto-resolution 2`, `--radiometric-calibration camera+sun`, `--pc-classify`.

---

## 2. Requirements for Our Farmland Map System

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent Layer                        │
│  (Orchestrates flights, triggers processing,            │
│   interprets results, generates farm recommendations)   │
└──────────────┬──────────────────────┬───────────────────┘
               │                      │
    ┌──────────▼──────────┐  ┌────────▼────────────────┐
    │   NodeODM Cluster   │  │  Agent Interpretation    │
    │  (Processing)       │  │  Service                 │
    │                     │  │  - VI analysis           │
    │  POST /task/new     │  │  - Anomaly detection     │
    │  GET /task/{id}/... │  │  - Yield estimation      │
    └──────────┬──────────┘  └────────┬────────────────┘
               │                      │
    ┌──────────▼──────────────────────▼────────────────┐
    │              Artifact Storage                      │
    │  (COG orthophotos, DSM/DTM, point clouds)         │
    │  S3-compatible or local filesystem                 │
    └──────────┬──────────────────────────────────────┘
               │
    ┌──────────▼──────────────────────────────────────┐
    │            Tile Server                            │
    │  (TiTiler or custom rio-tiler service)            │
    │  /tiles/{layer}/{z}/{x}/{y}.png                  │
    │  /metadata/{layer}                                │
    │  /algorithms/{vi_name}/{z}/{x}/{y}.png           │
    └──────────┬──────────────────────────────────────┘
               │
    ┌──────────▼──────────────────────────────────────┐
    │           Web Frontend (Map UI)                   │
    │  Leaflet/MapLibre + client-side VI rendering      │
    │  Contour overlays, field boundaries, annotations  │
    │  Agent insights panel (recommendations, alerts)   │
    └─────────────────────────────────────────────────┘
```

### 2.2 Processing Pipeline (REQ-PROC)

| ID | Requirement | Priority |
|---|---|---|
| PROC-01 | Submit drone images to NodeODM via `POST /task/new` with farmland-optimized options | P0 |
| PROC-02 | Poll `GET /task/{uuid}/info` for status; notify agent on completion | P0 |
| PROC-03 | Download COG orthophoto, DSM, and DTM from NodeODM on task completion | P0 |
| PROC-04 | Always pass `--cog --dsm --dtm --pc-classify` for farmland tasks | P0 |
| PROC-05 | Pass `--radiometric-calibration camera+sun` when multispectral imagery is detected | P0 |
| PROC-06 | Store artifacts in S3-compatible storage with project/field/date hierarchy | P1 |
| PROC-07 | Support NodeODM cluster (ClusterODM) for parallel field processing | P1 |
| PROC-08 | Extract and store processing metadata (GSD, bounds, band info) in our database | P1 |
| PROC-09 | Support incremental/partial re-processing when new images arrive | P2 |

### 2.3 Tile Serving (REQ-TILE)

| ID | Requirement | Priority |
|---|---|---|
| TILE-01 | Serve XYZ tiles on-the-fly from COG files (no pre-rendering required) | P0 |
| TILE-02 | Use **TiTiler** (or a rio-tiler based custom service) as tile server | P0 |
| TILE-03 | Support tile types: orthophoto (RGB/multispectral), DSM, DTM | P0 |
| TILE-04 | Provide TileJSON endpoint per layer for Leaflet/MapLibre integration | P0 |
| TILE-05 | Provide metadata endpoint returning band names, statistics, min/max per band | P0 |
| TILE-06 | Support colored hillshade rendering for DSM/DTM (gdaldem color-relief + hillshade merge) | P1 |
| TILE-07 | Support tile format options: PNG (default), JPEG, WebP | P1 |
| TILE-08 | Implement tile caching layer (Redis/CDN) for frequently accessed fields | P1 |
| TILE-09 | Support COG files on remote S3 storage (HTTP range requests) | P1 |
| TILE-10 | Serve individual bands as separate tile layers for client-side VI computation | P0 |

### 2.4 Vegetation Index Display (REQ-VI)

| ID | Requirement | Priority |
|---|---|---|
| VI-01 | Compute NDVI on-the-fly from multispectral orthophoto bands | P0 |
| VI-02 | Compute RGB-only indices (VARI, NGRDI, TGI, EXG) for standard camera imagery | P0 |
| VI-03 | Support both **server-side** VI tile rendering (TiTiler band math) and **client-side** (browser canvas) | P0 |
| VI-04 | Apply configurable color ramp (RdYlGn for health; custom for other indices) | P0 |
| VI-05 | Allow interactive histogram stretch (min/max adjustment) in the UI | P1 |
| VI-06 | Support NDRE and GNDVI for multispectral sensors with RedEdge band | P1 |
| VI-07 | Pre-compute and cache full-field VI rasters (GeoTIFF) for agent analysis | P0 |
| VI-08 | Provide zonal statistics API: mean/median/std VI per field zone or management unit | P0 |
| VI-09 | Generate temporal VI comparison (current vs. previous flight for same field) | P1 |
| VI-10 | Support radiometric calibration awareness (raw DN vs. reflectance flag in metadata) | P1 |

### 2.5 Contour & Elevation Analysis (REQ-ELEV)

| ID | Requirement | Priority |
|---|---|---|
| ELEV-01 | Generate contour lines on-demand from DSM/DTM with configurable interval | P1 |
| ELEV-02 | Return contours as GeoJSON for Leaflet overlay rendering | P1 |
| ELEV-03 | Use GDAL (`gdal_contour`) or GRASS GIS (`r.contour`) for contour generation | P1 |
| ELEV-04 | Support export of contours as Shapefile, GeoJSON, DXF | P2 |
| ELEV-05 | Generate hillshade visualization layer from DSM/DTM | P1 |
| ELEV-06 | Provide elevation query: point click returns elevation value from DSM/DTM | P1 |
| ELEV-07 | Support slope and aspect map generation for drainage analysis | P2 |
| ELEV-08 | Enable cross-section elevation profiles along a user-drawn line | P2 |

### 2.6 Map Frontend (REQ-MAP)

| ID | Requirement | Priority |
|---|---|---|
| MAP-01 | Interactive map with **Leaflet** or **MapLibre GL** with base layer switcher | P0 |
| MAP-02 | Display orthophoto as TMS layer with opacity control | P0 |
| MAP-03 | Toggle DSM/DTM colored hillshade overlay | P1 |
| MAP-04 | Toggle VI overlay with formula selector and color ramp | P0 |
| MAP-05 | Display contour lines as vector overlay | P1 |
| MAP-06 | Draw and manage field boundaries (polygons) with persistence | P0 |
| MAP-07 | Side-by-side or swipe comparison view (e.g., RGB vs NDVI, or date1 vs date2) | P1 |
| MAP-08 | Point inspector: click to see pixel values (elevation, VI, RGB, reflectance) | P1 |
| MAP-09 | Annotation/marker system for agent-generated alerts (pest, stress, irrigation) | P0 |
| MAP-10 | Export current map view as georeferenced image (PNG + world file) | P2 |
| MAP-11 | Timeline slider for multi-temporal datasets (same field, multiple dates) | P1 |

### 2.7 Agent Integration (REQ-AGENT)

| ID | Requirement | Priority |
|---|---|---|
| AGENT-01 | API for agents to trigger processing: upload images, set options, receive task ID | P0 |
| AGENT-02 | Webhook/event notification when processing completes | P0 |
| AGENT-03 | API for agents to request VI rasters and zonal statistics for a field polygon | P0 |
| AGENT-04 | API for agents to post map annotations (markers, polygons with metadata) | P0 |
| AGENT-05 | API for agents to request temporal change detection between two processing dates | P1 |
| AGENT-06 | Structured output format for agent analysis results (GeoJSON FeatureCollection with properties) | P0 |
| AGENT-07 | Agent can request specific band combinations or custom band math expressions | P1 |
| AGENT-08 | Agent can query point/zone elevation, slope, and aspect data | P1 |
| AGENT-09 | Agent recommendations displayed as a sidebar panel linked to map features | P0 |
| AGENT-10 | Audit trail: all agent-triggered actions and interpretations logged | P1 |

### 2.8 Data Management (REQ-DATA)

| ID | Requirement | Priority |
|---|---|---|
| DATA-01 | Project hierarchy: Organization > Farm > Field > Flight > Processing Result | P0 |
| DATA-02 | Store all COG artifacts with metadata (date, sensor, GSD, CRS, bounds) | P0 |
| DATA-03 | Support multi-temporal data per field (time series of flights) | P0 |
| DATA-04 | GeoPackage or PostGIS storage for field boundaries and vector data | P1 |
| DATA-05 | Retain processing options and NodeODM task configuration per result | P1 |

---

## 3. Recommended Technology Stack

| Component | Technology | Rationale |
|---|---|---|
| Processing | **NodeODM** (+ ClusterODM for scaling) | Direct ODM API, proven for drone imagery |
| Tile Serving | **TiTiler** (FastAPI + rio-tiler + cogeo-mosaic) | Purpose-built for COGs, supports band math, algorithms, mosaics |
| Artifact Storage | **S3-compatible** (MinIO or AWS S3) | TiTiler reads COGs via HTTP range requests |
| Map Frontend | **MapLibre GL JS** or **Leaflet** | MapLibre for WebGL perf; Leaflet for simpler plugin ecosystem |
| VI Rendering | **TiTiler band math API** (server) + **Canvas** (client fallback) | Server-side for agents; client-side for interactive adjustment |
| Contours | **gdal_contour** via API microservice | Lighter than GRASS, no session management needed |
| Backend API | **FastAPI** (Python) | Async, fast, pairs with TiTiler natively |
| Database | **PostgreSQL + PostGIS** | Spatial queries, field boundaries, metadata |
| Agent Orchestration | **Custom agent framework** | Trigger processing, interpret, annotate |
| Caching | **Redis** | Tile cache, task status, session state |

---

## 4. Key NodeODM Processing Options for Farmland

```json
{
  "dsm": true,
  "dtm": true,
  "cog": true,
  "orthophoto-resolution": 2,
  "dem-resolution": 5,
  "pc-classify": true,
  "radiometric-calibration": "camera+sun",
  "auto-boundary": true,
  "crop": 0,
  "fast-orthophoto": false,
  "skip-3dmodel": true
}
```

Notes:
- `--skip-3dmodel` saves time when only 2D analysis is needed
- `--radiometric-calibration camera+sun` ensures reflectance values for accurate VI computation
- `--cog` is essential for on-the-fly tile serving without pre-tiling
- `--pc-classify` enables ground classification for accurate DTM (drainage analysis)
- `--dem-resolution 5` gives 5cm DEM resolution for detailed contours

---

## 5. VI Computation Approaches (Decision Matrix)

WebODM uses **server-side only** (rio-tiler band math in Django). We should evaluate all three:

| Approach | Pros | Cons | When to Use |
|---|---|---|---|
| **Server-side tile math** (WebODM approach) | Consistent rendering, cacheable, formula library server-controlled, agent-accessible | Server CPU per tile, round-trip for parameter changes | Default display mode, agent API consumption |
| **Client-side canvas** | Instant histogram/rescale adjustment, no server round-trip | Limited to visible tiles, no full-field stats, JS complexity | Interactive fine-tuning after initial server render |
| **Pre-computed GeoTIFF** | Fastest serving, full-field raster for ML/analysis | Storage cost, stale if params change | Agent analysis, zonal stats, ML pipeline input |

**Recommendation:** Use TiTiler server-side band math as primary (matching WebODM's proven approach), with client-side canvas for interactive rescaling only (VI-05), and pre-computed GeoTIFFs for agent consumption (VI-07).

---

## 6. Implementation Phases

### Phase 1: Core Map Display (MVP)
- NodeODM integration (PROC-01 through PROC-04)
- TiTiler deployment with COG serving (TILE-01 through TILE-05, TILE-10)
- Map frontend with orthophoto display (MAP-01, MAP-02)
- Basic NDVI/VARI display (VI-01, VI-02, VI-04)
- Field boundary management (MAP-06)
- Agent processing trigger API (AGENT-01, AGENT-02)

### Phase 2: Analysis & Agent Intelligence
- Full VI suite with zonal statistics (VI-06 through VI-08, AGENT-03)
- Contour generation (ELEV-01 through ELEV-03)
- Hillshade and elevation query (ELEV-05, ELEV-06)
- Agent annotations on map (AGENT-04, AGENT-09)
- Temporal comparison (VI-09, MAP-07, MAP-11)

### Phase 3: Advanced Features
- Slope/aspect analysis (ELEV-07, ELEV-08)
- Multi-temporal change detection (AGENT-05)
- ClusterODM scaling (PROC-07)
- Export capabilities (MAP-10, ELEV-04)
- Custom band math expressions (AGENT-07)

---

## 7. Critical Integration Points

### 7.1 NodeODM to Tile Server Pipeline

```
NodeODM completes task
  → Download COG artifacts (orthophoto.tif, dsm.tif, dtm.tif)
  → Store in S3 with path: s3://bucket/{farm_id}/{field_id}/{date}/
  → Register in TiTiler mosaic (if combining multiple flights)
  → Extract metadata (bounds, bands, stats) → store in PostgreSQL
  → Notify agents of new data availability
```

### 7.2 Agent Analysis Flow

```
Agent receives "processing complete" event
  → Requests VI raster via TiTiler band math (full extent, low res)
  → Computes zonal statistics per management zone
  → Compares with historical data (temporal change)
  → Generates insights (stress zones, irrigation needs, growth stage)
  → Posts annotations to map API (GeoJSON features with recommendations)
  → Frontend displays agent insights panel + map markers
```

### 7.3 Multispectral Band Mapping

ODM stores multispectral bands in order: `[R, G, B, NIR, RedEdge, ...]`

For TiTiler band math, map band indices:
- Band 1 = Red, Band 2 = Green, Band 3 = Blue, Band 4 = NIR, Band 5 = RedEdge
- NDVI expression: `(b4 - b1) / (b4 + b1)`
- NDRE expression: `(b4 - b5) / (b4 + b5)`

For RGB-only cameras (3 bands):
- VARI expression: `(b2 - b1) / (b2 + b1 - b3)`
