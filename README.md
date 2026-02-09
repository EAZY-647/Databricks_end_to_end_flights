#  End-to-End Aviation Data Pipeline

### A Scalable Lakehouse Solution on Databricks

![Architecture Diagram] 
<img width="1408" height="752" alt="Gemini_Generated_Image_oipe9loipe9loipe" src="https://github.com/user-attachments/assets/fd662b6d-47cc-4a44-8228-4328ae899bc4" />


##  Project Overview

This project demonstrates a production-ready data engineering pipeline built using the **Databricks Lakehouse Platform**. The goal was to take messy, raw aviation data—simulated flight logs, passenger lists, and airport codes—and transform it into a clean, high-performance **Star Schema** optimized for analytics.

I built this to solve a common real-world problem: how to ingest data incrementally, clean it reliably, and model it for business intelligence tools like PowerBI or Tableau. The pipeline follows the industry-standard **Medallion Architecture** (Bronze $\rightarrow$ Silver $\rightarrow$ Gold), ensuring that every stage of the data is auditable and high-quality.

### Key Features
* **Incremental Ingestion:** Uses Databricks Autoloader to automatically detect and ingest new files as they arrive in cloud storage, preventing data duplication.
* **Data Quality Enforcement:** The "Silver" layer automatically handles schema drift, deduplicates records, and standardizes formats (e.g., converting timestamps and cleaning column names).
* **Dimensional Modeling:** The "Gold" layer implements a robust Star Schema with **SCD Type 1** (Slowly Changing Dimensions) logic, built using a custom dynamic Python script to handle upserts and surrogate key generation.

---

##  Architecture & Tech Stack

The solution is designed to be scalable and fault-tolerant, leveraging the best of modern data engineering tools.

| Component | Technology Used | Description |
| :--- | :--- | :--- |
| **Platform** | Databricks | The unified compute and storage engine. |
| **Storage** | Delta Lake | Provides ACID transactions, versioning, and time travel on top of the data lake. |
| **Ingestion** | Spark Structured Streaming | specifically **Autoloader**, for efficient, exactly-once file processing. |
| **Transformation** | PySpark & Delta Live Tables | For cleaning, joining, and aggregating data at scale. |
| **Modeling** | Custom Python Builder | A dynamic script I wrote to build Dimension and Fact tables programmatically. |

### Data Flow
1.  **🥉 Bronze Layer (Raw):** Raw JSON and CSV files are ingested from the landing zone into Delta tables. The schema is inferred automatically, and data is stored in its original state for auditability.
2.  **🥈 Silver Layer (Cleaned):** Data is cleaned, filtered, and standardized. Null values are handled, and "messy" column names are converted to snake_case for consistency.
3.  **🥇 Gold Layer (Curated):** The clean data is modeled into a Star Schema.
    * **Fact Table:** `fact_bookings` (Transactional data linked to dimensions).
    * **Dimensions:** `dim_flights`, `dim_passengers`, `dim_airports` (Descriptive data with surrogate keys).

---

## 📂 Repository Structure

Here is a guide to the files in this repo and what they do:

* **`0_project_setup.py`** *(formerly setup.ipynb)*:
    * Initializes the project environment.
    * Creates the necessary Catalog, Schemas (Databases), and Storage Volumes in Unity Catalog.
    
* **`1_bronze_ingestion.py`** *(formerly BronzeLayer.ipynb)*:
    * The ingestion engine. Configures Autoloader to stream data from the volume into Bronze tables (`bronze_flights`, `bronze_bookings`, etc.).
    
* **`2_silver_transformation.py`** *(formerly SilverNotebook.ipynb)*:
    * The transformation logic. Reads from Bronze, applies data quality rules (removing duplicates, formatting dates), and writes to the Silver layer.
    
* **`3_gold_dimensional_model.py`** *(formerly GOLD_DIMS.ipynb)*:
    * **The "Brain" of the project.** This notebook contains a custom-built Dimension Builder function.
    * Instead of writing repetitive SQL for each dimension, this script dynamically merges new data, generates surrogate keys (`monotonically_increasing_id`), and handles updates (SCD Type 1) for the `flights`, `passengers`, and `airports` dimensions.
    * It finally joins everything to create the `fact_bookings` table.

* **`config.py`** *(formerly SrcParameters.ipynb)*:
    * A central configuration file holding file paths, table names, and schema definitions to keep the code clean and DRY (Don't Repeat Yourself).

---

##  Results & Analytics

The final output is a queryable Data Warehouse. Below is an example of the kind of business insights this pipeline enables.

**Query: Total Revenue by Airline and Departure City**
*This query joins the Fact table with two Dimensions to aggregate revenue.*

```sql
SELECT 
    f.airline AS Airline_Name,
    a.city AS Departure_City,
    COUNT(b.booking_id) AS Total_Bookings,
    SUM(b.amount) AS Total_Revenue
FROM workspace.gold.fact_bookings b
JOIN workspace.gold.dim_flights f ON b.dim_flights_key = f.dim_flights_key
JOIN workspace.gold.dim_airports a ON b.dim_airports_key = a.dim_airports_key
GROUP BY f.airline, a.city
ORDER BY Total_Revenue DESC
LIMIT 5;
