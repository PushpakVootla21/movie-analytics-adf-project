# movie-analytics-adf-project

## 📑 Table of Contents

- [🎯 Objective](#-objective)
- [🔄 Data Flow & Transformations](#-data-flow--transformations)
- [🏗️ Architecture Overview](#️-architecture-overview)
- [📁 Project Structure](#-project-structure)
- [📥 Activity One: Data Ingestion](#-activity-one-data-ingestion)
- [🔄 Activity Two: Data Transformation Pipeline](#-activity-two-data-transformation-pipeline)
- [🛠️ Activity Three: Transformation Pipeline Orchestration](#-activity-three-transformation-pipeline-orchestration)
- [🚀 How to Run](#-how-to-run)

---

## 🎯 Objective

Analyze movie data for ABCD Company based on ratings using Azure Data Factory and Data Flow.


## 🔄 Data Flow & Transformations

1. **Archive Source File:**  
   After reading from the `raw` container, `archive` the original file.

2. **Column Rename:**  
  Change column name `'Rotton Tomato'` to `'Rotten Tomato'` for consistency.

3. **Drop Column:**  
   Remove the `Rating` column.

4. **Filter Movies:**  
   Only include movies made **after 1910 and before 2000**.

5. **Simplify Genres:**  
   Keep only the **first genre** listed per movie.

6. **Rank Movies:**  
   Determine each movie's rank within its **year and genre**.

7. **Group Analysis:**  
   For each **genre-year** group, calculate:
   - Average Rotten Tomatoes rating
   - Highest and lowest Rotten Tomatoes rating
   - Number of movies in the group

8. **Data Validation:**  
   If validation fails, route data to an **error** output.

9. **Routing Based on Year:**  
   - If movie year is **before 1950**: Move to `processed` folder in Blob Storage.
   - If movie year is **1950 or later**: Move to **SQL database**.

---

## 🏗️ Architecture Overview

- **Azure Data Factory** orchestrates the workflow.
- **Azure Data Flow** performs transformations.
- **Azure Blob Storage** stores raw, archived, and processed files.
- **Azure SQL Database** stores processed data for movies from 1950 onwards.

---

## 📁 Project Structure

```
movie-analytics-adf-project/
│
├── datafactory/
│   ├── pipelines/
│   │   ├── df_moviesdb_ingestion_pipeline.json           # Ingestion pipeline definition
│   │   └── df_movesdb_transform_pipeline.json            # Pipeline that executes the data flow
│   ├── datasets/
│   │   ├── ds_http_csv.json                             # HTTP (GitHub CSV) dataset
│   │   ├── ds_adls_csv.json                             # ADLS raw container dataset
│   │   ├── ds_adls_cleandatasink.json                   # ADLS processed/clean data dataset
│   │   └── AzureSqlTable1.json                          # Azure SQL Table dataset
│   └── linkedservices/
│       ├── ls_http_github.json                          # Linked service for GitHub HTTP
│       ├── ls_adls_gen2.json                            # Linked service for ADLS Gen2
│       └── ls_azuresql_db.json                          # Linked service for Azure SQL DB
│
├── dataflow/
│   └── transformations/
│       └── df_transform_moviesdb.json                   # Mapping Data Flow definition
│
├── docs/
│   ├── architecture.md                                  # Architecture documentation
│   └── architecture.png                                 # (Optional) Architecture diagram image
│   └── dataflow_transformation.png                      # (Optional) Data Flow transformation image
│
└── README.md                                            # Main project documentation
```
---

## 📥 Activity One: Data Ingestion

### Objective

Ingest the `moviesDB.csv` dataset from a public GitHub repository into the Azure Data Lake Storage (ADLS) raw container using Azure Data Factory.

### Steps

1. **Create Linked Services**
   - **HTTP Linked Service:**  
     In Azure Data Factory, create a new linked service of type HTTP.  
     - Set the base URL to the GitHub raw file URL:  
       `https://raw.githubusercontent.com/PushpakVootla21/Datasets/main/Files/moviesDB.csv`
   - **ADLS Gen2 Linked Service:**  
     Create a linked service for your Azure Data Lake Storage Gen2 account, pointing to your `raw` container.

2. **Create Datasets**
   - **Source Dataset:**  
     Create a dataset of type DelimitedText (CSV) using the HTTP linked service.  
     - Configure the dataset to read from the GitHub CSV file.
   - **Sink Dataset:**  
     Create a dataset of type DelimitedText (CSV) using the ADLS Gen2 linked service.  
     - Set the folder path to your `raw` container.

3. **Build the Ingestion Pipeline**
   - In Azure Data Factory, create a new pipeline (e.g., `df_moviesdb_ingestion_pipeline`).
   - Add a **Copy Data** activity:
     - **Source:** Select the HTTP CSV dataset.
     - **Sink:** Select the ADLS Gen2 CSV dataset.
     - Map columns as needed (usually auto-mapped if schema matches).

4. **Run the Pipeline**
   - Trigger the pipeline manually or schedule it as needed.
   - After execution, verify that `moviesDB.csv` appears in your ADLS `raw` container.

---

## 🔄 Activity Two: Data Transformation Pipeline

### Overview

This Mapping Data Flow (`df_transform_moviesdb`) processes the ingested movie data, applies business rules, and routes the results to different sinks (ADLS and SQL DB) based on the year and data quality.

### Step-by-Step Breakdown

#### 1. **Source**
- **Source Dataset:** Reads from the cleaned CSV in ADLS (`ds_adls_csv`).
- **Schema:**  
  - `movie` (integer)
  - `title` (string)
  - `genres` (string)
  - `year` (short)
  - `Rating` (short)
  - `Rotton Tomato` (short)
- **Options:**  
  - Allows schema drift and disables strict schema validation.
  - Moves files from `raw` to `archive` after reading.

#### 2. **Select Transformation**
- **Purpose:**  
  - Renames the column `{Rotton Tomato}` to `{Rotten Tomato}` for consistency.
  - Drops the `Rating` column (not mapped forward).
- **Result:**  
  - Output columns: `movie`, `title`, `genres`, `year`, `Rotten Tomato`.

#### 3. **Filter Transformation**
- **Purpose:**  
  - Filters movies to only include those made after 1910 and before 2000.
- **Expression:**  
  - `year > 1910 && year < 2000`

#### 4. **Derived Column Transformation**
- **Purpose:**  
  - Simplifies the `genres` column to keep only the first genre (splits by `|` and takes the first value).
- **Expression:**  
  - `genres = iif(instr(genres,'|') > 0, substring(genres, 1, instr(genres, '|')-1), genres)`

#### 5. **Window (Ranking) Transformation**
- **Purpose:**  
  - Ranks movies within each `year` and `genre` group based on `Rotten Tomato` rating (descending).
- **Output:**  
  - Adds a `movies_ranking` column.

#### 6. **Aggregate Transformation**
- **Purpose:**  
  - For each `genre` and `year` group, calculates:
    - Average Rotten Tomato rating (`avg_rotten_tomato`)
    - Maximum Rotten Tomato rating (`max_rotten_tomato`)
    - Minimum Rotten Tomato rating (`min_rotten_tomato`)
    - Total number of movies (`total_movies`)

#### 7. **Assert (Quality Check) Transformation**
- **Purpose:**  
  - Validates that the `year` is greater than 1920.
  - If validation fails, the row is flagged for error handling.

#### 8. **Split Transformation**
- **Purpose:**  
  - Splits the data into two streams:
    - `lessthan1950`: Movies with year < 1950
    - `greaterthan1950`: Movies with year >= 1950

#### 9. **Sink for Movies Before 1950**
- **Target:**  
  - Writes to ADLS (clean data sink) for movies before 1950.
  - Failed assertions are routed to an error folder.

#### 10. **Alter Row & Sink for Movies 1950 and After**
- **Alter Row:**  
  - Marks all rows for upsert (insert or update).
- **Target:** 
 - Writes to Azure SQL Database (`AzureSqlTable1`) for movies from 1950 onwards.

---
### Data Flow Diagram (Textual)

```
[Source: ADLS CSV]
      |
   [Select (Rename/Drop)]
      |
   [Filter (Year)]
      |
   [Derived Column (First Genre)]
      |
   [Window (Ranking)]
      |
   [Aggregate (Group Stats)]
      |
   [Assert (Year > 1920)]
      |
   [Split (<1950 / >=1950)]
      |                |
[Sink: ADLS <1950]  [Sink: SQL >=1950]
```

---

## 🛠️ Activity Three: Transformation Pipeline Orchestration

### Objective

Orchestrate the execution of the Mapping Data Flow (`df_transform_moviesdb`) using an Azure Data Factory pipeline.

### Steps

1. **Create the Pipeline**
   - In Azure Data Factory, create a new pipeline (e.g., `df_movesdb_transform_pipeline`).

2. **Add Execute Data Flow Activity**
   - Add an **Execute Data Flow** activity to the pipeline.
   - Set the referenced data flow to `df_transform_moviesdb`.

3. **Configure Data Flow Parameters**
   - If your data flow uses parameters (e.g., for sink dataset or schema), set them in the activity.  
     - Example: Set the `dbsink` parameter's schema to `movies`.

4. **Set Compute and Trace Options**
   - Assign compute resources (e.g., 8 cores, General compute type).
   - Set trace level to `Fine` for detailed logging.

5. **Run the Pipeline**
   - Trigger the pipeline manually or schedule it as needed.
   - Monitor execution and review logs for troubleshooting.

---

**Result:**  
This pipeline automates the execution of your transformation logic, ensuring that all business rules and data routing are applied to the ingested movie data.

### Key Points

- **Archiving:** Source files are archived after reading.
- **Column Standardization:** Ensures consistent naming (`Rotten Tomato`).
- **Data Quality:** Rows failing year validation are handled separately.
- **Routing:** Movies are routed to different sinks based on year.
- **Aggregation & Ranking:** Enables advanced analytics per genre and year.

## 🚀 How to Run

1. Make sure you have created the required linked services and datasets as described in the ingestion pipeline steps above.
2. Trigger the Data Factory ingestion pipeline to copy `moviesDB.csv` from GitHub to the `raw` container in Azure Blob Storage.
3. Once the file is ingested, trigger the transformation pipeline to process the data as per the business requirements.
4. Monitor pipeline runs in Azure Data Factory for any errors or warnings.
5. After successful execution, check the outputs in the processed folder (for movies before 1950) and in the SQL database (for movies from 1950 onwards).

---
## 📚 Further Reading

- [Azure Data Factory Linked Services](https://learn.microsoft.com/en-us/azure/data-factory/linked-services-introduction)
- [Azure Data Factory Datasets](https://learn.microsoft.com/en-us/azure/data-factory/concepts-datasets-linked-services)
- [Azure Data Factory Mapping Data Flows](https://learn.microsoft.com/en-us/azure/data-factory/concepts-data-flow-overview)