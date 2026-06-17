# Fabric Metadata-Driven ELT Framework

This project implements a **Metadata-Driven ELT Framework** on **Microsoft Fabric**. The architecture is designed to automate, scale, and easily manage Data Pipelines based on Metadata rather than hardcoding individual data workflows. 

This framework strictly follows the **Medallion Architecture** principles, logically separating data processing into distinct layers: from Source to Staging (Bronze), Staging to Transform (Silver), and Transform to Hub (Gold).

---

## 🏛️ Architecture Overview

The core architecture of the framework consists of 3 main components:

1. **Metadata Configuration:** Dictates *what* needs to be executed and *how*.
2. **Function Pipelines (Generic Logic):** Shared pipelines that execute the actual ELT (Extract, Load, Transform) logic.
3. **Data Subject Pipelines (Triggers):** Subject-specific pipelines acting as triggers to invoke the Function Pipelines.

### 1. Metadata Configuration Layer
Located in the `CONFIG_DATA/DATA/` directory, metadata (stored as `.csv` files or Lakehouse/SQL tables) is the heart of the system:
- **`elt_table_config.csv`**: Contains all definitions for tables requiring ETL processing. Key attributes include:
  - `sourcesystem`, `sourcetablename`: Source system and table information.
  - `layer`: The target data layer for the process (e.g., `src2bronze`, `bronze2silver`).
  - `file_type`, `stg_file_type`: Storage file formats (parquet, delta).
  - `full_refresh_flag`, `last_loaded_dt`, `criteria_columns`: Control flags to determine whether the pipeline performs a Full Load or an Incremental (Delta) Load.

Additionally, the `CONFIG_DATA/STORED_PROCEDURE/` directory (e.g., `update_load_status.sql`) contains procedures used to log execution details and update ELT process statuses.

### 2. Function Pipelines Layer
Located in the `FUNCTION_PIPELINES/` directory. Instead of creating hundreds of pipelines for hundreds of tables, the project relies on a few parameterized **Generic Pipelines**.
- **`coordinator.json`**: Acts as the Master Orchestrator, coordinating workflow execution.
- **`SRC2STG` (Source to Bronze/Staging):**
  - Reads the metadata configuration (`src2stg_get_config.json`).
  - Iterates through each targeted object (`src2stg_process_object.json`).
  - Copies data from the source system (e.g., SQL Server) into the Lakehouse/ADLS as Parquet files (`src2stg_copy_from_sql_to_parquet.json`).
- **`STG2TRANS` (Bronze to Silver/Transform):**
  - Reads configuration and iterates through staging objects.
  - Utilizes Spark Notebooks (`stg2trans_refresh_from_stg_nbk.ipynb`) to transform data from staging files into standardized Delta tables in the Silver Transform layer.
- **`TRANS2HUB` (Silver to Gold):**
  - Processes aggregation logic, handling dimensions and facts to push data into the Data Mart/Hub layer for reporting and analytics.

### 3. Data Subject Pipelines Layer
Located in the `DATASUBJECT_PIPELINES/` directory (e.g., `APPLICATION`).
- These pipelines (e.g., `src2stg_application_sqlvm_prd01_wf.json`) are extremely lean.
- They do not contain any data movement or transformation logic. Instead, they use the **InvokePipeline** activity (`CallToMainFunctionPipeline`) to pass execution context (Pipeline ID, Run ID, Trigger Time) to the **Function Pipelines**.
- This approach enables highly flexible permission management and scheduling based on specific Data Subjects.

---

## 🚀 Workflow Execution

1. **Trigger:** A schedule or manual trigger initiates a Data Subject Pipeline (e.g., `src2bronze_application_sqlvm_prd01_wf`).
2. **Invoke:** This pipeline calls the corresponding Generic Pipeline using the InvokePipeline feature.
3. **Get Config:** The Generic Pipeline reads the configuration file (`elt_table_config.csv`) and filters for tables belonging to the specific Data Subject (e.g., "application") and layer (e.g., "src2bronze").
4. **Process/Iterate:** A ForEach loop is activated to iterate over the filtered tables.
5. **ELT Execution:**
   - Data is either extracted (Copy Data Activity) or transformed (Spark Notebook execution).
   - The process automatically determines whether to perform a Full or Incremental load based on the `last_loaded_dt` and refresh flags.
6. **Audit/Log:** Upon completion, the pipeline executes the `update_load_status.sql` stored procedure to log the status (Success/Failed) and update the `last_loaded_dt` for the next run.

---

## 🌟 Architecture Benefits

- **Scalability:** Need to onboard 10 new tables? There is no need to create 10 new pipelines. Simply add 10 configuration rows to `elt_table_config.csv`.
- **Maintainability:** Processing logic (e.g., Copy Logic or Spark Transformations) is centralized in `FUNCTION_PIPELINES`. Update in one place to apply globally.
- **Easy Audit & Tracking:** Every pipeline run is recorded using metadata control tables and stored procedures.

---

## 🛠️ Technologies Used
- **Platform:** Microsoft Fabric (Data Factory Pipelines, Synapse Data Engineering / Spark Notebooks, Lakehouse).
- **Data Architecture:** Medallion Architecture (Bronze, Silver, Gold).
- **Data Formats:** Parquet, Delta Tables.
- **Languages:** PySpark (Notebooks), SQL (Stored Procedures), JSON (Pipelines).
