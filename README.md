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

