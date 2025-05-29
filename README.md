# 🚀 Data Ingestion: Azure Data Factory + ADLS Gen2

### 📊 Data Pipeline Overview

![Pipeline Flow](DataIngestionSteps&screenshots/pipeline.png)

This step focuses on ingesting raw data from two sources:

1. 📦 **GitHub (HTTP CSV files)** — multiple files
2. 📓 **Clever Cloud MySQL (olist\_order\_payments)** — single SQL table

All ingested data is stored in the **Bronze layer** of **ADLS Gen2** for further processing.

---

## 📁 Folder Structure

```bash
olistsample/
├── bronze/
│   ├── olist_customers_dataset.csv              (from GitHub HTTP)
│   ├── olist_geolocation_dataset.csv            (from GitHub HTTP)
│   ├── olist_order_items_dataset.csv            (from GitHub HTTP)
│   ├── olist_order_reviews_dataset.csv          (from GitHub HTTP)
│   ├── olist_orders_dataset.csv                 (from GitHub HTTP)
│   ├── olist_products_dataset.csv               (from GitHub HTTP)
│   ├── olist_sellers_dataset.csv                (from GitHub HTTP)
│   └── olist_order_payments.csv                 (from Clever Cloud SQL)
```


## ✅ Ingestion Pipeline Overview

Azure Data Factory pipeline handles both CSV and SQL ingestion:

* Uses **ForEach loop** for batch ingesting GitHub CSVs
* Uses a direct **Copy Activity** for MySQL table ingestion

---

## ⚙️ Step-by-Step Instructions

### 1️⃣ Create Pipeline Parameter

* Go to pipeline > Parameters tab
* Add parameter:

  * **Name**: `ForEachInput`
  * **Type**: `Array`
  * **Default value**:

```json
[
  {
    "csv_relative_url": "data/olist_customers_dataset.csv",
    "file_name": "olist_customers_dataset.csv"
  },
  {
    "csv_relative_url": "data/olist_orders_dataset.csv",
    "file_name": "olist_orders_dataset.csv"
  }
]
```

---

### 2️⃣ Add `ForEach` Activity

* From Activities > Iteration > drag `ForEach` into canvas
* Set **Items** = `@pipeline().parameters.ForEachInput`
* Enable **Sequential execution**

---

### 3️⃣ Inside `ForEach` → Add `Copy data` Activity for CSVs

* Add `Copy data` activity inside `ForEach`
* **Source Dataset**: `DataFromGithubViaLinkedService`

  * Type: HTTP
  * Linked service:

    * Base URL: `https://raw.githubusercontent.com/Raviteja000-maker/`
    * Authentication type: Anonymous
  * Set **Relative URL** as: `@dataset().csv_relative_url`
* **Sink Dataset**: `CsvFromLinkedServiceToSink`

  * Linked service: ADLS Gen2
  * File path: `olistdata/bronze/@dataset().file_name`

---

### 4️⃣ Add `Copy data` Activity (Outside `ForEach`) for MySQL Table

* This handles the SQL table (olist\_order\_payments) separately
* **Source Dataset**: `MySqlTable1`

  * Linked service: MySQL

    * Server name: `bcsc8smql9mx8vfjkwn-mysql.services.clever-cloud.com`
    * Port: `3306`
    * Database: `bcsc8smql9mx8vfjkwn`
    * Username: `ulg3fxgahqhu244y`
    * Password: `Your Password`
    * Table: `olist_order_payments`
* **Sink Dataset**: same ADLS Gen2 container

  * File path: `olistdata/bronze/olist_order_payments.csv`

---

## 💡 Notes

* Ensure ADLS Gen2 linked service has correct storage account/container (`olistdata`)
* Enable "First row as header" in sink if CSVs have headers
* Don't forget to **publish** the pipeline before triggering

---

## 🔄 Expected Outcome

* GitHub CSVs ➔ copied into ADLS Gen2 under `bronze/`
* SQL table `olist_order_payments` ➔ also copied to `bronze/`

---

## 📊 Next Step

## 🔄 Enhancing Pipeline with Dynamic JSON via Lookup (GitHub)

To improve modularity and make your pipeline **easier to maintain**, you've removed the hardcoded JSON from the parameter input and replaced it with a **dynamic Lookup** pointing to a JSON file hosted on GitHub.

### ❌ Why Not Parameterize JSON Inline?

Initially, the pipeline used a static JSON inside the ForEach parameter array:

```json
[
  {
    "csv_relative_url": "olist_orders_dataset.csv",
    "file_name": "olist_orders_dataset.csv"
  },
  ...
]
```

But this method is:

* Not scalable when new files are added
* Hard to maintain
* Requires code changes to update input definitions

---

### ✅ Using Lookup + GitHub JSON

Instead of inline JSON, you now:

1. **Host a JSON file on GitHub**
   Example: `https://raw.githubusercontent.com/Raviteja000-maker/azure-etl-pipeline-with-databricks/refs/json_file_path.json`

2. **Create a new Dataset (HTTP – JSON)**

   * Linked to GitHub via HTTP
   * Authentication: Anonymous
   * Used in a Lookup Activity

3. **Use Lookup to feed ForEach input**

   * In `ForEach > Settings > Items`, set:

     ```
     @activity('LookForeachInput').output.value
     ```

4. ✅ Now your pipeline automatically picks up new files defined in the GitHub JSON.

---

### 🔁 Updated Pipeline Flow

```text
Lookup (GitHub JSON)
     ↓
ForEach (Loop over GitHub CSVs)
     ↓
Copy Activity
```

* `olist_order_payments.csv` from **Clever Cloud MySQL** still uses a direct copy.
* GitHub-based CSVs are now controlled via the JSON lookup.

---

### 🌐 Benefits of This Approach

* 🔧 **Easier to update** – no pipeline redeploys for file changes
* 📁 **Clean separation** between pipeline logic and metadata
* 🌍 **Remote configurability** via GitHub

## 📊 Next Step

# Databricks to ADLS Gen2 Access via Azure App Registration (OAuth2)

Now we are going to enable secure access from Databricks to Azure Data Lake Storage Gen2 using OAuth 2.0 via Azure Active Directory App Registration.

---

## 1. Register an Application in Azure AD

1. Go to **Azure Portal > Azure Active Directory > App registrations**.
2. Click on **"New registration"**.
3. Provide a name like `olist-app-registration-db-adls`.
4. Choose **Supported account type** as `My organization only`.
5. Click **Register**.

---

## 2. Create Client Secret

1. Go to the newly created app.
2. Navigate to **Certificates & secrets > Client secrets**.
3. Click **"New client secret"**, set expiration, and click **Add**.
4. **Copy and save** the secret value (not just the secret ID).

---

## 3. Assign RBAC Role on ADLS Gen2

1. Go to the **Storage Account** (e.g., `olistecommstore`).
2. Go to **Access Control (IAM) > Role Assignments**.
3. Click **"+ Add > Add role assignment"**.
4. Select Role: `Storage Blob Data Contributor`.
5. In Members tab, assign access to:

   * **App Registration** (search by name: `olist-app-registration-db-adls`).
6. Click **Review + assign**.

---

## 4. Configure Spark in Databricks

In your Databricks notebook, define the following variables and set Spark configurations:

```python
storage_account = "olistecommstore"
application_id = "<Application (client) ID>"
service_credential = "<Client Secret>"
directory_id = "<Directory (tenant) ID>"

spark.conf.set(f"fs.azure.account.auth.type.{storage_account}.dfs.core.windows.net", "OAuth")
spark.conf.set(f"fs.azure.account.oauth.provider.type.{storage_account}.dfs.core.windows.net", \
    "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set(f"fs.azure.account.oauth2.client.id.{storage_account}.dfs.core.windows.net", application_id)
spark.conf.set(f"fs.azure.account.oauth2.client.secret.{storage_account}.dfs.core.windows.net", service_credential)
spark.conf.set(f"fs.azure.account.oauth2.client.endpoint.{storage_account}.dfs.core.windows.net", \
    f"https://login.microsoftonline.com/{directory_id}/oauth2/token")
```

---

## 5. Access Data from ADLS Gen2

```python
file_path = "abfss://<container>@olistecommstore.dfs.core.windows.net/<folder>/<file>.csv"

df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load(file_path)

display(df)
```

---

By registering an Azure AD app and assigning it the necessary RBAC role, you can securely access ADLS Gen2 from Databricks using OAuth2 credentials without storing access keys.

---

**Note:** Ensure that the secret value, app ID, and tenant ID are stored securely in Azure Key Vault or Databricks secrets for production use.

## 🔄 Data Transformation & Enrichment in Databricks (Post Ingestion)

After ingesting data from multiple sources (HTTP API, SQL DB) into **Azure Data Lake Storage Gen2 (Bronze Layer)**, we proceed with transformation, enrichment, and writing to the **Silver Layer** using Databricks.

---

### 📥 Step 1: Read Raw Data from ADLS Gen2 (Bronze Layer)

We use the `abfss://` protocol to read CSV files stored in the Bronze container:

```python
base_path = "abfss://olist@<storage_account>.dfs.core.windows.net/Bronze/"
orders_df = spark.read.format("csv").option("header", "true").load(base_path + "olist_orders_dataset.csv")
payments_df = spark.read.format("csv").option("header", "true").load(base_path + "olist_order_payments_dataset.csv")
... (read all datasets similarly)
```

---

### 🛠 Step 2: Transform and Join Data

Once the raw data is ingested from various sources like HTTP endpoints, SQL Server, and MongoDB, we move on to processing and transformation within Databricks. This stage ensures that the data is clean, enriched, and analytics-ready before being written to the Silver layer.

Key transformation tasks include:

* **Data Cleaning**: Identify and handle missing or null values across datasets to maintain integrity.
* **Type Casting**: Convert string columns representing dates or timestamps into appropriate datetime formats.
* **Column Standardization**: Rename or standardize column names for consistency across datasets.
* **Joins**: Combine multiple datasets (orders, payments, customers, products, sellers) using common keys like `order_id`, `customer_id`, and `product_id`.
* **Enrichment**: Merge additional metadata or translated fields from MongoDB into the primary datasets.
* **Basic Validation**: Perform sanity checks on joined data (row counts, duplicates, schema alignment).

After these transformations, the curated data is saved in **Parquet format** under the **Silver layer** in Azure Data Lake Storage Gen2 (ADLS Gen2), ready for efficient querying and downstream analytics.

### 🌍 Step 3: Enrich with MongoDB (Product Category Translation)

We fetch translated category names from a MongoDB collection using `pymongo`:

```python
from pymongo import MongoClient
import pandas as pd

uri = "mongodb://<user>:<password>@<host>:27018/<database>"
client = MongoClient(uri)
db = client["<database>"]
collection = db["product_category_translations"]
translations_df = pd.DataFrame(list(collection.find()))
```

Convert to Spark DataFrame and join:

```python
translations_df["_id"] = translations_df["_id"].astype(str)
translations_spark_df = spark.createDataFrame(translations_df)
final_df = final_df.join(translations_spark_df, "product_category_name", "left")
```

---

### 📦 Step 4: Write Enriched Data to ADLS Gen2 (Silver Layer)

We write the resulting enriched dataset in Parquet format for optimized storage and querying:

```python
final_df.write.mode("overwrite").parquet("abfss://olist@<storage_account>.dfs.core.windows.net/Silver/")
```

This will write the data in partitioned files (e.g., `part-0000*`) into the Silver container.

---

*  Raw data is read from ADLS Gen2 (Bronze).
*  Multiple datasets are joined in Spark.
*  Translation enrichment from MongoDB is added.
*  Final enriched data is written as **Parquet** files to the **Silver** layer in ADLS Gen2.

This silver data can now be used for analytical workloads, reporting, or further processing in a gold layer.

➡️ *Next step: Create external views in Synapse using OPENROWSET to query Parquet data directly.*

## 🧾 Synapse Analytics Flow: External Table Creation from ADLS Gen2

This guide outlines the complete process of querying and writing Parquet data from ADLS Gen2 using Azure Synapse Analytics. It covers schema creation, credentials, external sources, views, and writing results to another layer.

---

### ✅ Step 1: Set up File Format & Credential

```sql
CREATE EXTERNAL FILE FORMAT extfileformat 
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
);

CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword@123';

CREATE DATABASE SCOPED CREDENTIAL synapse_identity
WITH IDENTITY = 'Managed Identity';
```

---

### 🔗 Step 2: Register ADLS Gen2 as External Data Source

```sql
CREATE EXTERNAL DATA SOURCE olist_adls
WITH (
    LOCATION = 'https://olistecommstore.dfs.core.windows.net/',
    CREDENTIAL = synapse_identity
);
```

---

### 🏷 Step 3: Create Schema and View (Reading from Silver Layer)

```sql
CREATE SCHEMA gold;

CREATE VIEW gold.final2 AS
SELECT *
FROM OPENROWSET(
    BULK 'olist/Silver/',
    DATA_SOURCE = 'olist_adls',
    FORMAT = 'PARQUET'
) AS result
WHERE order_status='delivered'
```

---

### 📋 Step 4: Read from the View

```sql
SELECT * FROM gold.final2;
```

---

### 📂 Step 5: Create External Data Source for Gold Layer

This is where you want to save enriched/processed results.

```sql
CREATE EXTERNAL DATA SOURCE goldlayer 
WITH (
    LOCATION = 'https://olistecommstore.dfs.core.windows.net/olist/Gold/',
    CREDENTIAL = synapse_identity
);
```

---

### 🪄 Step 6: Write Output from Gold View to Final Table

```sql
CREATE EXTERNAL TABLE gold.finaltable 
WITH (
    LOCATION = 'finalServing',
    DATA_SOURCE = goldlayer,
    FILE_FORMAT = extfileformat
)
AS
SELECT * FROM gold.final2;
```

Note: `gold.final2` must be another view or staging table, if you're referring to a transformed output or filtered results.

---

### 📌 Summary

* Data is first queried from ADLS Gen2 (Silver path) using OPENROWSET.
* That data is materialized as a View inside Synapse.
* A new Data Source pointing to the Gold folder is created.
* The final external table writes enriched/filtered view results into ADLS Gen2 (Gold).

---

### 🔗 Reference:

[Microsoft Docs - CREATE EXTERNAL TABLE AS SELECT (CETAS)](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/develop-tables-cetas)



