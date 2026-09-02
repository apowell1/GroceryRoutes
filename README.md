# Grocery Route Optimization

A two-stage vehicle routing and itinerary optimization pipeline using Google Places API, OpenRouteService Matrix API, Haversine spatial filtering, and Google OR-Tools Constraint Solver to find the optimal grocery shopping trip subject to time budgets and transit modes.

---

## Purpose & Overview

Planning multi-stop grocery shopping trips across specialized stores (organic grocers, discount supermarkets, ethnic markets, wholesale clubs) is a variant of the **Travelling Salesperson Problem with Optional Visited Nodes and Time Window Constraints (TSP with Prize Collecting / Service Penalties)**.

This project automates the entire routing pipeline:
1. **Candidate Store Discovery**: Queries the Google Places API for target grocery chains and markets across a designated county.
2. **Geocoding & Spatial Indexing**: Geocodes origin/destination addresses and builds a pairwise Haversine distance matrix across all candidate store locations.
3. **Stage 1 Candidate Reduction & Travel-Time Matrix**: Prunes distant locations to fit OpenRouteService API matrix dimension limits and fetches exact driving/walking duration matrices.
4. **Stage 1 Routing Optimization**: Formulates and solves a routing problem in Google OR-Tools with store shopping service times (`SERVICE_TIME`) and node drop penalties (`SKIP_PENALTY`).
5. **Corridor Expansion (Stage 2)**: Dynamically expands the candidate set around transition corridors along the initial route and re-solves for the global optimum.
6. **Itinerary & Mapping**: Generates a detailed turn-by-turn itinerary and interactive Folium route map.

---

## Tech Stack & Dependencies

- **Language & Environment**: Python 3 / Jupyter Notebook (`GroceryStoreRoutes.ipynb`)
- **Optimization**:
  - `ortools` (`pywrapcp`, `routing_enums_pb2` — Google Operations Research Tools constraint solver)
  - `protobuf`
- **Mapping & Geocoding APIs**:
  - `googlemaps` (Google Places text search and geocoding)
  - `openrouteservice` (driving and walking travel-time duration matrices)
  - `geopy` (`Nominatim` fallback geocoder)
- **Spatial Geometry & Geospatial Analysis**:
  - `haversine` (spherical distance computations)
  - `shapely` (`Point`, `LineString` corridor buffers)
  - `folium` (interactive Leaflet maps)
- **Data Wrangling & Utilities**:
  - `pandas`, `numpy`, `requests`, `tqdm`

---

## Repository Structure

```
.
└── GroceryStoreRoutes.ipynb   # Main Jupyter notebook implementing the two-stage routing optimization
```

---

## Workflow & Algorithm Architecture

```
User Parameters (Start/End Address, Time Budget, Travel Mode, Store List)
        │
        ▼
Google Places API Search (County Boundary & Target Store Chains)
        │
        ▼ (Stores DataFrame with Lat/Lng & Place IDs)
Haversine Distance Matrix & Candidate Filtering
        │
        ▼
OpenRouteService API (Duration Matrices: Driving / Walking)
        │
        ▼
OR-Tools Routing Model (Stage 1 TSP with Drop Penalties & Time Limit)
        │
        ▼ Initial Route Selection
Spatial Corridor Buffer & Candidate Expansion (Stage 2)
        │
        ▼ Recomputed ORS Matrices
Final OR-Tools Optimization (Stage 2)
        │
        ▼
Optimized Itinerary & Folium Interactive Map
```

---

## Setup & Configuration

### 1. Prerequisites & Dependencies

Install required libraries:

```bash
pip install ortools openrouteservice googlemaps geopy shapely haversine folium pandas numpy requests tqdm
```

### 2. API Keys

Ensure valid API keys are provided in Cell 7 of the notebook:
- `GOOGLE_MAPS_API_KEY`: Requires Places API and Geocoding API enabled.
- `ORS_API_KEY`: OpenRouteService API key for matrix distance and duration endpoints.

### 3. User Parameters

Configure route parameters in `GroceryStoreRoutes.ipynb`:

| Parameter | Type | Description |
|---|---|---|
| `START_ADDRESS` / `END_ADDRESS` | string | Trip origin and destination addresses |
| `TOTAL_HOURS` | float | Total time budget for travel + shopping (e.g. `1.5` hours) |
| `TRAVEL_MODE` | string | `"walking"`, `"driving"`, or `"fastest"` |
| `SERVICE_TIME` | float | Fixed shopping duration per visited store in minutes (default: `10.0`) |
| `COUNTY` | string | Search boundary (e.g. `"San Diego County, California"`) |
| `STORES` | list[str] | Target grocery brands/stores to query |

---

## Execution

Launch the notebook via Jupyter:

```bash
jupyter notebook GroceryStoreRoutes.ipynb
```

Run all cells sequentially to retrieve candidate locations, solve the optimization stages, and display the final route itinerary.

---

## Key Modules & Optimization Parameters

- **`haversine(coord1, coord2, unit=Unit.MILES)`**: Fast spatial distance filtering for initial candidate pruning.
- **`pywrapcp.RoutingIndexManager` & `pywrapcp.RoutingModel`**: OR-Tools routing graph solver with time dimension tracking (`AddDimension`).
- **`SKIP_PENALTY` (`1440` minutes)**: Disincentivizes skipping stores while respecting the upper bound `TOTAL_HOURS` constraint.
- **`RoutingSearchParameters`**: Employs `PATH_CHEAPEST_ARC` first solution strategy and `GUIDED_LOCAL_SEARCH` metaheuristic with configurable search time limits (`TIME_LIMIT_1`, `TIME_LIMIT_2`).

---

## Limitations & Project Status

- Requires active Google Maps and OpenRouteService API credentials and internet connectivity for live matrix queries.
- API rate limits apply to OpenRouteService matrix endpoints (managed via candidate pruning and sleep intervals).
