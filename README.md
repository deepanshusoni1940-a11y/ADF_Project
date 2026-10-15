ADF_Project

# Azure Data Factory End -To-End Project 

An end-to-end data engineering project built in **Azure Data Factory** that ingests flight-booking data from multiple sources, cleans and models it in a **bronze / silver / gold** lakehouse on **ADLS Gen2**, and serves analytics-ready aggregates.

## Architecture

```
 On-prem CSV files (self-hosted IR) ──┐
 REST API (JSON) ─────────────────────┼──►  BRONZE  ──►  SILVER (cleaned, Delta)  ──►  GOLD (aggregated)
 SQL database (incremental, Parquet) ─┘
```

| Layer | What it contains | How it is built |
|-------|------------------|-----------------|
| **Bronze** | Raw data landed as-is in the `bronze` container: `onprem/` (CSV), `github/` (JSON from the API), `sql/` (Parquet), `monitor/lastload/` (watermark file) | Copy activities in `onprem_ingestion`, `API_Ingestion` and `SQLToDatalake` |
| **Silver** | Cleaned, conformed dimension and fact tables stored as **Delta** | `SilverLayer` pipeline running the `DataTransformation` mapping data flow |
| **Gold** | Business-ready aggregates, e.g. total sales and ranking per airline | `GoldLayer` pipeline running the `DataServing` mapping data flow |

## Pipelines

| Pipeline | Purpose |
|----------|---------|
| `onprem_ingestion` | Copies CSV files from an on-prem machine through a **self-hosted integration runtime**. A parameterised source/sink dataset and a **ForEach** loop over a file list handle many files with one Copy activity. |
| `API_Ingestion` | Pulls JSON data from a REST API into `bronze/github`. |
| `SQLToDatalake` | **Incremental load** from SQL using a **watermark** pattern: Lookup activities read the last and latest load values, a Copy activity moves only new rows into `bronze/sql` as Parquet, and a final Copy updates the watermark file in `bronze/monitor/lastload`. |
| `SilverLayer` | Runs the `DataTransformation` data flow to build the silver tables. |
| `GoldLayer` | Runs the `DataServing` data flow to build the gold tables. |
| `parent pipeline` | Orchestrates the pipelines above end to end. |

## Transformations (Mapping Data Flows)

**Silver (`DataTransformation`)** builds `DimAirline`, `DimFlight`, `DimPassenger`, `DimAirport` and `FactBookings`:
- Derived columns, column selection, type casting (e.g. cost)
- Filters and derived name/gender fields for passengers
- **Alter Row** with upsert logic and **Delta merge** sinks, so reruns update existing rows instead of duplicating them

**Gold (`DataServing`)**:
- Joins bookings with the airline dimension
- Aggregates total sales per airline
- Ranks airlines with a **Window** transformation (`denseRank()`)

# Key concepts demonstrated

- Medallion (bronze/silver/gold) architecture on ADLS Gen2
- Parameterised datasets and pipelines, ForEach loops
- Incremental loading with watermarks
- Self-hosted integration runtime for on-prem sources
- Mapping data flows, Delta Lake merge/upsert
- Failure alerting with **Azure Logic Apps** (HTTP trigger + email), called from a Web activity
- Source control with **GitHub** (`main` collaboration branch, `adf_publish` for ARM templates)

# Lessons learned

- A Delta merge fails with "multiple source rows matched" when the source has duplicate keys. Deduplicate before the sink (Aggregate or Window + Filter) and make sure the sink key columns are unique.
- Window ranking only works across rows when the `Over` partition is empty and a sort is set.
- Dynamic content must pass a single value (e.g. `@item().file_name`), not a whole object or array.

# Tech stack

Azure Data Factory, Azure Data Lake Storage Gen2, Delta Lake, SQL, Mapping Data Flows, Azure Logic Apps, GitHub

## Author

**Deepanshu Soni**
