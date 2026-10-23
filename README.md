# Weather-Demographics-Data-pipeline
An end-to-end automated data pipeline that pulls live weather metrics for 50 global capital cities from the Open-Meteo API, merges them with static population demographics using Python and Pandas, loads the cleaned dataset into Google BigQuery, transforms it with SQL, and visualizes the results in an interactive Looker Studio map dashboard.

Built as the practical assignment for the Technical Foundational Training Bootcamp (Linux, GitHub, Python, GCP, SQL).

## Architecture

```
Linux setup -> GitHub repo -> Python ETL -> BigQuery table -> SQL logical view -> Looker Studio
```

| Stage | Tooling | Output |
|---|---|---|
| 1. Local setup | Linux / Git Bash | Project directory structure |
| 2. Version control | Git, GitHub | `feature/data-ingestion` branch with logical commits |
| 3. Python ETL | `requests`, `pandas` | Merged, cleaned DataFrame |
| 4. Cloud ingestion | `google-cloud-bigquery` | `current_weather_stats` table |
| 5. SQL logic | BigQuery SQL | `vw_hottest_large_countries` view |
| 6. Dashboard | Looker Studio | Interactive geo map with category filter |

## Project Structure

```
weather_pipeline/
├── data/                    # Local CSV payloads (gitignored)
│   └── population.csv
├── scripts/
│   ├── main.py              # ETL script (extract, transform, load)
│   └── gcp-key.json         # Service account key (gitignored, never committed)
├── sql/
│   ├── 01_explore.sql       # Top 10 rows and row count
│   ├── 02_top5_hottest.sql  # Top 5 hottest countries over 10M population
│   └── 03_create_view.sql   # Logical view for reporting
├── .gitignore
├── requirements.txt
└── README.md
```

## Data Sources

- **Weather:** [Open-Meteo Forecast API](https://open-meteo.com/) (`current_weather=true`), queried by latitude and longitude for each capital city. No API key required.
- **Demographics:** `population.csv` containing Country, City, Latitude, Longitude and Population for 50 capital cities.

## Prerequisites

- Python 3.10 or higher
- A personal Google account with a GCP project named `weather-demographics-bootcamp`
- BigQuery API enabled
- A service account with the **BigQuery Admin** role and a downloaded JSON key
- A BigQuery dataset named `global_weather_data`

## Setup

1. Clone the repository:
   ```bash
   git clone https://github.com/<your-username>/weather_pipeline.git
   cd weather_pipeline
   ```

2. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate        # Windows Git Bash: source venv/Scripts/activate
   ```

3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

4. Add the data file. Place `population.csv` inside `data/`.

5. Add your credentials. Save your service account key as `scripts/gcp-key.json`.

   > **Security:** `gcp-key.json`, `data/`, `.env`, virtual environments and `__pycache__/` are all listed in `.gitignore`. Never commit credentials.

## Usage

Run the pipeline from the project root:

```bash
python scripts/main.py
```

The script will:

1. Load `population.csv` into a Pandas DataFrame.
2. Call the Open-Meteo API for each capital and build a weather DataFrame.
3. Merge both DataFrames on `Country`.
4. Check for nulls, drop rows with missing temperature or population, and impute missing wind speed with the median.
5. Enforce data types (`Population` as `int64`, `Temperature_C` as `float64`).
6. Derive `Temperature_F` and `Temperature_Category`.
7. Load the result into BigQuery as `current_weather_stats` using `WRITE_TRUNCATE` (the table is replaced on each run).

### Temperature categories

| Category | Rule |
|---|---|
| Cold | Below 10 °C |
| Mild | 10 °C to 25 °C |
| Hot | Above 25 °C |

## BigQuery

| Object | Name |
|---|---|
| Dataset | `global_weather_data` |
| Table | `current_weather_stats` |
| View | `vw_hottest_large_countries` |

### Table schema

| Column | Type | Description |
|---|---|---|
| Country | STRING | Country name |
| City | STRING | Capital city |
| Latitude | FLOAT | Latitude of the capital |
| Longitude | FLOAT | Longitude of the capital |
| Population | INTEGER | Country population |
| Temperature_C | FLOAT | Current temperature in Celsius |
| Windspeed_kmh | FLOAT | Current wind speed in km/h |
| Observation_Time | TIMESTAMP | Time of the weather observation |
| Temperature_F | FLOAT | Temperature in Fahrenheit |
| Temperature_Category | STRING | Cold, Mild or Hot |

### SQL scripts

Replace `YOUR_PROJECT_ID` in each file with your GCP project ID, then run them in the BigQuery console in order:

1. `sql/01_explore.sql` validates the schema and row count.
2. `sql/02_top5_hottest.sql` returns the top 5 hottest countries with a population over 10 million, using `GROUP BY`, `ORDER BY` and aggregate functions.
3. `sql/03_create_view.sql` creates `vw_hottest_large_countries`, the semantic layer used by the dashboard.

## Dashboard

The Looker Studio report connects to `vw_hottest_large_countries` and includes:

- A **Geo map** with Country as the location and average temperature as the color gradient
- A **drop-down filter** for Temperature Category (Cold, Mild, Hot)
- A summary table of Country, Population and temperature
- A custom theme and descriptive headers

**Live dashboard:https://datastudio.google.com/reporting/d35de177-2ce8-441e-8101-616a43b68479
