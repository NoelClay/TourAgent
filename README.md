# TourAgent — Seoul Weather Data Pipeline

**Team project (AI-TOMIC)** · Python · Data Engineering · 2025

A geospatial weather data pipeline that collects real-time forecast data for Seoul's top 120 tourist attractions from the Korea Meteorological Administration (KMA) API. Built as the data layer for a LangGraph-based RAG travel recommendation agent.

---

## What This Does

```
Seoul 120 attractions (shapefile)
        │
        ▼
  Centroid computation (GeoPandas)
        │
        ▼
  Nearest KMA grid point (nx, ny)   ← Euclidean distance matching
        │
        ▼
  KMA Forecast API call per location  ← 단기예보 (VilageFcstInfoService_2.0)
        │
        ▼
  weather_data_all.csv              → RAG agent context
```

---

## My Contribution

I built the entire data pipeline (`scripts/`):

| Module | What it does |
|--------|-------------|
| `weather_utils.py` | Calculate latest KMA update time (API updates every 3 hours on fixed schedule) |
| `geo_processor.py` | Load `.shp` shapefile for 120 attractions, compute polygon centroids, match each to nearest KMA grid (nx, ny) via Euclidean distance |
| `weather_api.py` | Batch-fetch short-term forecast data from KMA REST API with retry logic and rate limiting |
| `main.py` | Orchestrate the full pipeline → output CSV |

**Key design decisions:**

- **Coordinate system**: Attractions are stored in Korean TM (EPSG:5179) shapefile. Reprojected to WGS84 (EPSG:4326) before matching against KMA's lat/lon grid.
- **Grid matching**: KMA's forecast grid uses integer (nx, ny) tile coordinates. Each attraction maps to the nearest tile via Euclidean distance over the merged GeoDataFrame.
- **Rate limiting**: 0.5s sleep between API calls + up to 5 retries per location to handle KMA's unstable endpoints.
- **Null handling**: KMA occasionally returns empty or malformed responses for certain regions. All such cases are logged and skipped without crashing the pipeline.

---

## Architecture (Full System)

This repo is the **data ingestion layer** of a larger travel agent:

```
TourAgent Pipeline (this repo)
  └─ weather_data_all.csv
        │
        ▼
Django Backend
  └─ LangGraph RAG Agent
        ├─ Vector store (tourist attraction embeddings)
        ├─ Weather context injection
        └─ Travel recommendation generation
```

The RAG agent retrieves attractions relevant to a user query, enriches them with current weather from this pipeline's output, and generates recommendations.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Geospatial | GeoPandas 1.0, Shapefile (ESRI), EPSG:4326/5179 |
| Data | pandas, numpy, openpyxl |
| API | KMA 단기예보 REST API (VilageFcstInfoService_2.0) |
| Config | python-dotenv |

---

## Setup

```bash
pip install -r requirements.txt

# Set KMA API key
echo "WEATHER_API_DECODING=<your-key>" > .env

# Required data files in data/raw/:
# - 서울시 주요 120장소 영역.shp (+ .dbf, .prj, .shx)
# - 기상청41_단기예보 조회서비스_오픈API활용가이드_격자_위경도.xlsx
# - 서울시 주요 120장소 목록.xlsx

cd scripts && python main.py
# Output: data/processed/weather_data_all.csv
```

---

Original organization repo: [AI-TOMIC/TourAgent](https://github.com/AI-TOMIC/TourAgent)
