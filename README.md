# 🚀 Data Ingestion: Azure Data Factory + ADLS Gen2

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

Proceed to the **Data Transformation** step using **Azure Databricks**.
