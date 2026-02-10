# Module 3 Homework: Data Warehousing & BigQuery for Data Engineering Zoomcamp 2026

In this homework we'll practice working with BigQuery and Google Cloud Storage.

When submitting your homework, you will also need to include
a link to your GitHub repository or other public code-hosting
site.

This repository should contain the code for solving the homework.

When your solution has SQL or shell commands and not code
(e.g. python files) file format, include them directly in
the README file of your repository.

## Data

For this homework we will be using the Yellow Taxi Trip Records for January 2024 - June 2024 (not the entire year of data).

Parquet Files are available from the New York City Taxi Data found here:

https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

## Loading the data

You can use the following scripts to load the data into your GCS bucket:

- Python script: [load_yellow_taxi_data.py](./load_yellow_taxi_data.py)
- Jupyter notebook with DLT: [DLT_upload_to_GCP.ipynb](./DLT_upload_to_GCP.ipynb)

You will need to generate a Service Account with GCS Admin privileges or be authenticated with the Google SDK, and update the bucket name in the script.

If you are using orchestration tools such as Kestra, Mage, Airflow, or Prefect, do not load the data into BigQuery using the orchestrator.

Make sure that all 6 files show in your GCS bucket before beginning.

Note: You will need to use the PARQUET option when creating an external table.

### Answer

#### 1. Install `uv` to create a project

```
$ curl -LsSf https://astral.sh/uv/install.sh | sh
```

#### 2. Create a project

```bash
$ uv init
```

#### 3. Add dependencies

```bash
$ uv add google-api-python-client
$ uv add google-cloud-storage
```

#### 4. Run the loading script

```bash
$ uv run load_yellow_taxi_data.py
```

## BigQuery Setup

Create an external table using the Yellow Taxi Trip Records.

Create a (regular/materialized) table in BQ using the Yellow Taxi Trip Records (do not partition or cluster this table).

### Answer

#### 1. Create the External Table

```sql

CREATE OR REPLACE EXTERNAL TABLE `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_ext`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://this-is-my-bucket-name/yellow_tripdata_2024-*.parquet']
);
```

#### 2. Create the Regular (Non-Partitioned) Table

```sql
CREATE OR REPLACE TABLE `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned` AS
SELECT * FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_ext`;
```

## Question 1. Counting records

What is count of records for the 2024 Yellow Taxi Data?
- 65,623
- 840,402
- 20,332,093
- 85,431,289

### Answer

```sql
SELECT count(*) FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`;
```

> **Results:** $20,332,093$

## Question 2. Data read estimation

Write a query to count the distinct number of PULocationIDs for the entire dataset on both the tables.

What is the **estimated amount** of data that will be read when this query is executed on the External Table and the Table?

- 18.82 MB for the External Table and 47.60 MB for the Materialized Table
- 0 MB for the External Table and 155.12 MB for the Materialized Table
- 2.14 GB for the External Table and 0MB for the Materialized Table
- 0 MB for the External Table and 0MB for the Materialized Table

### Answer

```sql
SELECT COUNT(DISTINCT PULocationID)
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`;
```

> **Results:** $262$ distinct PULocationIDs; with $155,12$ MB on Materialized Table and $0$ MB on External Table

## Question 3. Understanding columnar storage

Write a query to retrieve the PULocationID from the table (not the external table) in BigQuery. Now write a query to retrieve the PULocationID and DOLocationID on the same table.

Why are the estimated number of Bytes different?
- BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires
reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.
- BigQuery duplicates data across multiple storage partitions, so selecting two columns instead of one requires scanning the table twice,
doubling the estimated bytes processed.
- BigQuery automatically caches the first queried column, so adding a second column increases processing time but does not affect the estimated bytes scanned.
- When selecting multiple columns, BigQuery performs an implicit join operation between them, increasing the estimated bytes processed

### Answer

#### 1. Test query #1

```sql
SELECT PULocationID
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`;
```

#### 2. Test query #2

```sql
SELECT PULocationID, DOLocationID
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`;
```

> **Results:** BigQuery is a Columnar Store. This means it stores data by columns, not rows. If you select 2 columns, it has to read 2 independent chunks of data. If you select 1, it only reads 1.

## Question 4. Counting zero fare trips

How many records have a fare_amount of 0?
- 128,210
- 546,578
- 20,188,016
- 8,333

### Answer

```sql
SELECT COUNT(*)
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`
WHERE fare_amount = 0;
```

> **Results:** $8333$

## Question 5. Partitioning and clustering

What is the best strategy to make an optimized table in Big Query if your query will always filter based on tpep_dropoff_datetime and order the results by VendorID (Create a new table with this strategy)

- Partition by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on tpep_dropoff_datetime Partition by VendorID
- Partition by tpep_dropoff_datetime and Partition by VendorID

### Answer

Select the option that Partitions by the filter column and Clusters by the ordering column:

* **Partitioning:** Breaks a large table into smaller physical pieces (partitions). This is best used for high-cardinality filtering (like dates). If you partition by tpep_dropoff_datetime, BigQuery can skip reading entire days/months of data that don't match your filter.

* **Clustering:** Sorts the data within each partition. This helps with further filtering and sorting/ordering.

> **Results:** "Partition by tpep_dropoff_datetime and Cluster on VendorID"

## Question 6. Partition benefits

Write a query to retrieve the distinct VendorIDs between tpep_dropoff_datetime
2024-03-01 and 2024-03-15 (inclusive)

Use the materialized table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 5 and note the estimated bytes processed. What are these values?

Choose the answer which most closely matches.

- 12.47 MB for non-partitioned table and 326.42 MB for the partitioned table
- 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table
- 5.87 MB for non-partitioned table and 0 MB for the partitioned table
- 310.31 MB for non-partitioned table and 285.64 MB for the partitioned table

### Answer

#### 1. Create the test table

```sql
CREATE OR REPLACE TABLE `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_partitioned_clustered`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS
SELECT * FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_ext`;
```

#### 2. Compare query #1

```sql
SELECT DISTINCT VendorID
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`
WHERE tpep_dropoff_datetime >= '2024-03-01' AND tpep_dropoff_datetime <= '2024-03-15';
```

#### 3. Compare query #2

```sql
SELECT DISTINCT VendorID
FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_partitioned_clustered`
WHERE tpep_dropoff_datetime >= '2024-03-01' AND tpep_dropoff_datetime <= '2024-03-15';
```

> **Results:** $310.24$ MB for non-partitioned table and $26.84$ MB for the partitioned table

## Question 7. External table storage

Where is the data stored in the External Table you created?

- Big Query
- Container Registry
- GCP Bucket
- Big Table

### Answer

> **Results**: GCP Bucket

## Question 8. Clustering best practices

It is best practice in Big Query to always cluster your data:
- True
- False

### Answer

> **Results:** **False** because tiny tables (e.g., < 1GB) overhead of managing clusters help if the table is so small that scanning the whole thing takes milliseconds anyway

## Question 9. Understanding table scans

No Points: Write a `SELECT count(*)` query FROM the materialized table you created. How many bytes does it estimate will be read? Why?

### Answer

This was my query:

```sql
SELECT count(*) FROM `this-is-my-project-id.this-is-my-dataset-id.yellow_tripdata_2024_non_partitioned`;
```

It should be 0 B. Because BigQuery keeps metadata about tables, including the total row count. It doesn't need to scan the actual data rows to answer COUNT(*).
