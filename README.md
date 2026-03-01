Toronto Shelter System — Public Services (Shelter Occupancy)

A data analysis + prediction notebook for the Toronto shelter system public services dataset.
We clean and aggregate the data, run exploratory Awareness analysis, build an occupancy-rate prediction model for Spreadness, evaluate it (including walk-forward validation + ranking metrics), and generate a small 2-day simulation (Dec 31, 2025 actuals → Jan 1, 2026 predictions).

----------------------------------------------------------------

1) Dataset

Source (CSV): public_services_dataset.csv hosted on GitHub (loaded via raw URL)

Main columns used:
- OCCUPANCY_DATE (date)
- LOCATION_POSTAL_CODE (postal code)
- SECTOR (Men/Women/Families/Youth/Mixed Adult)
- OVERNIGHT_SERVICE_TYPE
- PROGRAM_MODEL
- PROGRAM_AREA
- CAPACITY_TYPE
- ACTUAL_CAPACITY
- OCCUPIED_CAPACITY
- UNAVAILABLE_CAPACITY (used in Awareness; sometimes omitted in Spreadness model)
- OCCUPANCY_RATE (computed)

----------------------------------------------------------------

2) Environment / Dependencies

Install (if missing):
- pandas, numpy, scikit-learn, plotly
Optional (mapping):
- geopandas, requests, folium
Optional (dashboard):
- streamlit, streamlit-folium

Example install command:
pip install pandas numpy scikit-learn plotly geopandas folium requests streamlit streamlit-folium

----------------------------------------------------------------

3) Pipeline Overview

We run two main blocks:

A) First Analysis — Awareness
Goal: understand who / where / when / what drives high occupancy.

Steps:
1. Load data from GitHub URL
2. Clean postal codes (strip/uppercase/remove spaces)
3. Drop rows with missing LOCATION_POSTAL_CODE or missing PROGRAM_MODEL
4. Validate postal code format using regex
5. Convert postal codes to FSA (first 3 characters) for consistent geography
6. Aggregate daily totals at: (OCCUPANCY_DATE, FSA, SECTOR, CAPACITY_TYPE)
7. Visualize:
   - Occupancy by Sector (bar chart)
   - Top crowded FSAs (hotspots)
   - Seasonality by month (line chart)
   - Occupancy by Program Area (bar chart)
8. Optional: mapping
   - Download FSA boundaries (StatCan ArcGIS endpoint)
   - Merge with occupancy stats
   - Build Folium choropleth
   - Streamlit app for interactive selection

Outputs:
- Plots for 2025 sector / area / seasonality analysis
- GeoJSON boundary file: fsa_boundary_M_and_L4L.geojson
- Aggregated data export: fsa_2025.csv
- Streamlit app: app.py

----------------------------------------------------------------

B) Second Analysis — Spreadness
Goal: predict occupancy patterns and evaluate whether the model can support “load balancing” decisions.

B1) Data Preparation
1. Validate postal code format using regex
2. Drop invalid postal codes (or keep valid-only subset for modeling)
3. Aggregate to a stable unit so the model isn’t confused by duplicate rows:

Group by:
- OCCUPANCY_DATE
- LOCATION_POSTAL_CODE
- SECTOR
- OVERNIGHT_SERVICE_TYPE
- PROGRAM_MODEL
- PROGRAM_AREA
- CAPACITY_TYPE

Aggregate:
- Sum ACTUAL_CAPACITY
- Sum OCCUPIED_CAPACITY

Then recompute:
- OCCUPANCY_RATE = OCCUPIED_CAPACITY / ACTUAL_CAPACITY

Add calendar features:
- month
- day_of_week

B2) Encoding
We encode categorical columns into integer IDs:
- LOCATION_POSTAL_CODE
- SECTOR
- PROGRAM_AREA
- PROGRAM_MODEL
- OVERNIGHT_SERVICE_TYPE
- CAPACITY_TYPE

New columns created:
- *_encoded

B3) Baseline
Baseline predictor:
- Sector mean occupancy rate (baseline_pred = mean(OCCUPANCY_RATE | SECTOR))

Metric:
- MAE (mean absolute error)

B4) Model
Model:
- HistGradientBoostingRegressor

Key hyperparameters:
- learning_rate = 0.05
- max_iter = 200
- categorical_features = [0,1,2,3,4,5] (encoded categorical columns)

Features used:
- LOCATION_POSTAL_CODE_encoded
- SECTOR_encoded
- PROGRAM_AREA_encoded
- PROGRAM_MODEL_encoded
- OVERNIGHT_SERVICE_TYPE_encoded
- CAPACITY_TYPE_encoded
- month
- day_of_week

Target:
- OCCUPANCY_RATE

Guardrail:
- clip predictions into [0, 1]

B5) Evaluation (Advanced)
1) Walk-forward validation (4 folds)
Train on early blocks, test on the next block, repeated 4 times.
Report per fold:
- MAE, RMSE, R²
- clip rate

2) Error slicing
- MAE by SECTOR
- MAE by CAPACITY_TYPE
- MAE by near-full status (OCCUPANCY_RATE >= 0.95 vs not)

3) Ranking evaluation for routing (Top-K)
Because load balancing is a ranking problem (pick least-full shelters), we evaluate:
For each day:
- choose Top-K lowest predicted occupancy
- compare to Top-K lowest true occupancy

Metrics:
- Top-K capture: fraction overlap with true best K
- Regret: how much worse the recommended K is vs oracle K (lower is better)

We report results for:
- K = 10
- K = 25

4) Feature importance
Permutation importance (MAE increase when feature is shuffled) to identify what drives predictions.

----------------------------------------------------------------

4) Simulation / Prediction Output (Dec 31 → Jan 1)

We generate a 2-day table:
- 2025-12-31: use actual occupancy rate from existing data
- 2026-01-01: generate predictions using the trained model

Steps:
1. Extract actual rows for 2025-12-31
2. Create “entries” table of unique program definitions (no date)
3. Cross-join entries with 2026-01-01
4. Predict occupancy rate for 2026-01-01
5. Combine the two days and sort
6. Drop non-paired entries (must have both dates)

Output:
- predicted_occupancy_2026.csv (paired rows only)

----------------------------------------------------------------

5) Notes / Known Limitations
- The model uses only static IDs + calendar features, so it tends to predict a stable “typical” occupancy for each program/location.
- Without lag/rolling time-series features, day-to-day fluctuations are hard to capture.
- Occupancy is heavily skewed toward near-full; the model may look good overall but perform worse on rare “has-space” cases.

----------------------------------------------------------------

6) How to Run
1) Run the notebook top-to-bottom:
- Data load + cleaning
- Awareness analysis charts (optional)
- Spreadness model training + evaluation
- Simulation output generation

2) Optional: run Streamlit map:
streamlit run app.py

----------------------------------------------------------------

7) Outputs Produced
- fsa_boundary_M_and_L4L.geojson (FSA boundaries for GTA)
- fsa_2025.csv (aggregated occupancy by FSA/sector/capacity type)
- app.py (Streamlit choropleth app)
- predicted_occupancy_2026.csv (Dec 31 actual + Jan 1 predicted, paired rows only)

----------------------------------------------------------------

8) Author / Team
Datathon project — Toronto Shelter System analysis and occupancy prediction.
