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

## Mathematical Formulation & Operations Research Methodology

The pipeline formalizes grocery trip planning as an **Orienteering Problem with Soft Penalty / Prize-Collecting Travelling Salesperson Problem (PCTSP)** with cumulative time dimension constraints and metric space embedings:

### 1. Great-Circle Spherical Haversine Metric
For initial spatial candidate clustering on the earth sphere of mean radius \(R \approx 3958.8\text{ miles}\), the geodesic distance between \(\mathbf{x}_1 = (\phi_1, \lambda_1)\) and \(\mathbf{x}_2 = (\phi_2, \lambda_2)\) is:
\[
d_H(\mathbf{x}_1, \mathbf{x}_2) = 2R \arcsin \left( \sqrt{\sin^2\left(\frac{\Delta\phi}{2}\right) + \cos\phi_1 \cos\phi_2 \sin^2\left(\frac{\Delta\lambda}{2}\right)} \right)
\]
This spatial metric satisfies the triangle inequality \(d_H(u, v) \le d_H(u, w) + d_H(w, v)\), allowing candidate pruning without expensive non-Euclidean road network graph queries.

### 2. Mixed-Integer Program Formulation (Prize-Collecting TSP with Time Budget)
Let \(G = (V, E)\) be a complete directed graph where:
- \(V = \{0\} \cup \{1, \dots, n\} \cup \{n+1\}\), where \(0\) is the origin depot, \(n+1\) is the destination depot, and \(V_{\text{stores}} = \{1, \dots, n\}\) are candidate grocery stores.
- \(c_{ij} \ge 0\) is the transit duration between node \(i\) and node \(j\) (derived from the OpenRouteService matrix).
- \(s_i \ge 0\) is the fixed shopping service time at store \(i\) (\(s_0 = s_{n+1} = 0\)).
- \(p_i > 0\) is the penalty for omitting store \(i\) (`SKIP_PENALTY`).
- \(T_{\text{max}}\) is the total elapsed time budget (`TOTAL_HOURS \times 60`).

Decision variables:
- \(x_{ij} \in \{0, 1\}\): \(1\) if the vehicle travels directly from node \(i\) to node \(j\).
- \(y_i \in \{0, 1\}\): \(1\) if store \(i \in V_{\text{stores}}\) is visited (\(y_0 = y_{n+1} = 1\)).
- \(t_i \ge 0\): Arrival timestamp at node \(i\).

**Objective Function**:
\[
\min \sum_{i \in V} \sum_{j \in V} c_{ij} x_{ij} + \sum_{i \in V_{\text{stores}}} p_i (1 - y_i)
\]
**Subject to**:
- **Flow Conservation**:
  \[
  \sum_{j \in V \setminus \{0\}} x_{0j} = 1, \quad \sum_{i \in V \setminus \{n+1\}} x_{i, n+1} = 1
  \]
  \[
  \sum_{j \in V \setminus \{i\}} x_{ij} = y_i, \quad \sum_{j \in V \setminus \{i\}} x_{ji} = y_i, \quad \forall i \in V_{\text{stores}}
  \]
- **Subtour Elimination & Arrival Time Propagation (Miller-Tucker-Zemlin constraints)**:
  \[
  t_j \ge t_i + s_i + c_{ij} - M(1 - x_{ij}), \quad \forall i, j \in V, \, j \ne 0
  \]
- **Time Budget Cap**:
  \[
  t_{n+1} \le T_{\text{max}}
  \]

### 3. Metaheuristic Search Strategy (OR-Tools)
Because the Prize-Collecting TSP is \(\mathcal{NP}\)-hard, Google OR-Tools solves the system using:
- **First Solution Heuristic**: `PATH_CHEAPEST_ARC` (greedy insertion minimizing marginal transit cost).
- **Local Search Metaheuristic**: `GUIDED_LOCAL_SEARCH` (penalizes frequently visited local minima edges to escape suboptimal basins of attraction within time limits `TIME_LIMIT_1` and `TIME_LIMIT_2`).

### 4. Two-Stage Spatial Corridor Dilation
In Stage 2, candidate density is refined along the convex hull corridor \(\mathcal{C}\) between active route vertices \(u^*, v^*\):
\[
\text{Buffer}(\mathcal{C}, \delta) = \left\{ \mathbf{x} \in \mathbb{R}^2 \mid \inf_{\mathbf{y} \in \overline{u^* v^*}} \|\mathbf{x} - \mathbf{y}\|_2 \le \delta \right\}
\]
allowing local detours without suffering full combinatorial explosion over the entire county.

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
