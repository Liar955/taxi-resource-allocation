# NYC Taxi Spatio-Temporal Demand Modeling

This project studies NYC yellow taxi passenger demand as an uncertain spatio-temporal count process. Using January 2026 yellow taxi trip records and TLC taxi zone geometries, the project characterizes demand patterns across space and time, formulates stochastic demand models, forecasts future pickup demand, and evaluates whether spatial dependence improves predictive performance.

The current scope focuses on demand-side modeling. Supply-demand optimization is discussed as future work because the public TLC trip records do not directly observe real-time taxi supply, vacant vehicle availability, or dispatch decisions.

## Project Motivation

Urban taxi demand is highly uneven across regions and time periods. Some areas, such as Midtown Manhattan and airport zones, show persistent high demand, while other zones exhibit sparse or highly variable demand. For transportation planning and future fleet allocation, it is important to understand:

- where demand is spatially concentrated,
- whether demand has strong daily or weekly periodicity,
- how uncertain future demand is,
- whether neighboring-zone demand helps improve forecasts.

This project builds a demand-side modeling pipeline that can later serve as input to resource allocation or dispatch optimization models.

## Research Questions

1. What are the spatial and temporal structures of NYC yellow taxi pickup demand?
2. Does demand exhibit periodicity across hours of day and days of week?
3. Can pickup demand be modeled as a stochastic count process?
4. Does the Poisson/NHPP variance assumption hold for zone-time taxi demand?
5. Can quantile forecasting capture demand uncertainty?
6. Does adding spatial lag information improve forecasting over temporal-only models?

## Data

The project uses:

- NYC TLC Yellow Taxi Trip Records, January 2026
- NYC TLC Taxi Zone shapefile

Key trip-level fields include:

- `tpep_pickup_datetime`
- `tpep_dropoff_datetime`
- `PULocationID`
- `DOLocationID`
- `trip_distance`
- `fare_amount`
- `total_amount`
- `passenger_count`

The original raw data is not included in this repository due to file size and reproducibility considerations. Users should download the trip records and taxi zone files from the NYC TLC data portal and place them in the expected local data directory.

## Methodology

### 1. Data Preprocessing

The raw trip records are cleaned by removing invalid or implausible trips:

- non-positive trip distance,
- non-positive fare amount,
- missing pickup zone,
- invalid trip duration,
- records outside January 2026.

Pickup timestamps are converted into fixed time intervals. The main analysis uses 30-minute bins.

### 2. Demand Characterization

The project first explores demand structure through:

- spatial demand intensity maps,
- hourly demand profiles,
- weekday-hour heatmaps,
- zone-hour demand patterns,
- daily demand anomaly heatmaps,
- zone-level demand heterogeneity diagnostics.

Demand intensity is defined as:

```text
intensity_z = pickup_count_z / area_z
```

This is used for visualization of spatial concentration. It is not the main forecasting target.

### 3. Zone-Time Panel Construction

The central modeling unit is a zone-time cell:

```text
(PULocationID, time_bin)
```

For each zone `z` and time interval `t`, demand is defined as:

```text
Y_{z,t} = number of pickups in zone z during interval t
```

The forecasting target is next-interval demand:

```text
Y_{z,t+1} = target_next_pickup_count
```

Features include:

- cyclic time features: `hour_sin`, `hour_cos`, `weekday_sin`, `weekday_cos`,
- lag demand features: `pickup_lag_1`, `pickup_lag_2`, `pickup_lag_4`, `pickup_lag_48`,
- rolling demand features: `rolling_mean_2h`, `rolling_std_2h`, `rolling_mean_6h`,
- dropoff-based proxy features,
- aggregated trip attributes.

### 4. Stochastic Demand Modeling

The project begins with a Poisson/NHPP baseline. In continuous time, pickups can be viewed as arrivals from a non-homogeneous Poisson process with time-varying intensity. After aggregation into zone-time intervals, this implies:

```text
Y_{z,t+1} | X_{z,t} ~ Poisson(mu_{z,t})
```

The Poisson model assumes:

```text
E[Y | X] = Var[Y | X]
```

The project then diagnoses overdispersion using variance-to-mean behavior and out-of-sample Pearson dispersion. If the equidispersion assumption fails, a Negative Binomial extension is used:

```text
Y_{z,t+1} | X_{z,t} ~ Negative Binomial(mu_{z,t}, alpha)
```

with:

```text
Var[Y | X] = mu + alpha * mu^2
```

For scalability, the full-panel Poisson baseline is estimated using `sklearn` with sparse zone one-hot encoding. Full-panel performance is evaluated using out-of-sample metrics rather than AIC.

### 5. Machine Learning Forecasting

The project uses gradient boosting models to forecast next-interval pickup demand. The ML model serves as the predictive layer of the project, complementing the interpretable stochastic count models.

Evaluation metrics include:

- MAE,
- RMSE,
- sMAPE,
- mean Poisson deviance.

### 6. Quantile Demand Uncertainty

To estimate demand uncertainty, the project trains quantile forecasting models for:

```text
q10, q50, q90
```

The uncertainty width is defined as:

```text
uncertainty_width = pred_q90 - pred_q10
```

This is used as a model-based measure of future demand uncertainty. Wider intervals indicate zone-time cells where demand is less predictable or more variable.

### 7. Spatial Dependence Modeling

The project constructs a taxi-zone adjacency matrix using zone geometries. Neighboring-zone demand is summarized through spatial lag features:

```text
spatial_lag_pickup_{z,t} = sum_j W_{z,j} Y_{j,t}
```

where `W_{z,j}` is a spatial weight between zones.

The project compares:

- temporal-only forecasting models,
- spatio-temporal forecasting models with spatial lag features.

This comparison evaluates whether spatial dependence improves demand prediction.

## Scope and Future Work

This version focuses on demand-side modeling. A full supply-demand optimization model is not included in the main analysis because the dataset does not contain real-time taxi availability, vacant vehicle locations, driver supply, or dispatch decisions.

A future extension could combine the demand forecasts from this project with observed or simulated vehicle supply data. Predicted demand quantiles such as `pred_q50` and `pred_q90` could then be used for average-case and robust vehicle allocation models.

## Repository Structure

Recommended structure:

```text
taxi-resource-allocation/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── nyc_taxi_demand_modeling.ipynb
├── data/
│   ├── raw/
│   └── processed/
├── figures/
├── reports/
└── src/
```

The current main analysis notebook is:

```text
new_nyc taxi or.ipynb
```

For a cleaner GitHub repository, it is recommended to rename or copy it to:

```text
notebooks/nyc_taxi_demand_modeling.ipynb
```

## Environment

Core Python packages:

```text
pandas
numpy
geopandas
matplotlib
seaborn
scikit-learn
statsmodels
scipy
pyarrow
jupyter
```

## How to Run

1. Download the NYC TLC yellow taxi trip records and taxi zone shapefile.
2. Place the data files in the local `data/raw/` directory or update paths in the notebook.
3. Install dependencies:

```bash
pip install -r requirements.txt
```

4. Open the notebook:

```bash
jupyter notebook
```

5. Run the notebook from top to bottom.

## Key Takeaways

- NYC taxi demand is highly spatially concentrated and temporally periodic.
- Airport and Manhattan zones show persistent high baseline demand.
- Zone-time demand is a sparse, overdispersed count process.
- Poisson/NHPP provides a natural stochastic baseline, but overdispersion motivates a Negative Binomial extension.
- Quantile forecasting provides a practical measure of demand uncertainty.
- Spatial lag features allow the project to test whether neighboring-zone demand improves forecasting.

