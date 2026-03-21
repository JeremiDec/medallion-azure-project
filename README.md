# medallion-azure-project

# NYC Taxi Data Processing - Medallion Architecture

## Project Overview
This project implements a scalable data pipeline using **Azure Databricks** and **Azure Blob Storage** to process 15GB of raw NYC Yellow Taxi trip data (covering years 2014-2024). The architecture follows the **Medallion design pattern** (Bronze, Silver, Gold) to transform raw, inconsistent data into high-value business insights.

## Tech Stack
- **Compute:** Azure Databricks (Runtime 14.3 LTS, Apache Spark 3.5.0)
- **Storage:** Azure Blob Storage (WASBS Protocol)
- **Language:** PySpark, Spark SQL, Python
- **Security:** Databricks Secrets API for credential masking

## Data Architecture
The data flows through three distinct layers:
1. **Bronze (Raw):** 15GB of Parquet files ingested directly from NYC Open Data.
2. **Silver (Cleaned):** Data processed to ensure quality. Applied filters: 
   - `fare_amount > 0`
   - `passenger_count > 0`
   - Chronological validation (`dropoff > pickup`)
   - Removal of duplicates.
3. **Gold (Curated):** Aggregated business-level data. Joined facts with `taxi_zones` lookup table to analyze revenue by borough.

## Technical Challenges & Solutions (Data Quality Risks)
- **Schema Evolution Crisis:** Encountered a `ClassCastException` due to physical data type changes in Parquet files across different years (INT64 vs DOUBLE). 
  - *Solution:* Implemented an iterative union process with explicit schema enforcement and manual casting during ingestion.
- **Large Scale Processing:** Handling 150M+ records (15GB) required disabling the Vectorized Parquet Reader to maintain type flexibility.
- **Security:** Integrated Databricks Secrets to prevent leaking Azure Access Keys in the public repository.

## Business Insights
The final SQL analysis identifies the most profitable taxi zones in New York City, providing insights into total revenue, trip counts, and average trip distances per borough.

## How to Reproduce
1. Configure a Databricks Cluster (Single User mode recommended).
2. Set up a Secret Scope named `azure-storage` with your Azure Key.
3. Run the notebooks in order: `01_Ingestion`, `02_Silver_Transformation`, `03_Gold_Analysis`.