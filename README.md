# Azure Banking Data Platform 

A production-grade, real-time data engineering pipeline built using **Azure Event-Driven Architecture**, **Cosmos DB**, and the **Databricks Lakehouse Platform**.

---

# 1. Project Overview

This solution delivers an end-to-end banking data platform capable of handling **real-time ATM and UPI transactions**, **customer profile updates**, **KYC records**, and **branch performance analytics**.

The platform integrates:

* **Event-driven ingestion** using Event Grid, Azure Functions, and Service Bus
* **Operational data store** using Azure Cosmos DB
* **Data Lakehouse ETL** using Databricks (Bronze → Silver → Gold)
* **SCD-2 dimension management**
* **SQL Data Warehouse serving layer**
* **Power BI dashboards for Customer 360, Fraud Metrics, Branch Performance**
* **CI/CD workflow using GitHub Actions / Azure DevOps**

---

# 2. High-Level Architecture

```
+------------------+          +-----------------+         +----------------+
|   ADLS Raw       |  Event   |  Azure Function |  Msg    |   Service Bus  |
| (ATM/UPI/KYC)    +--------->+      (A)        +-------->+     Queue      |
+------------------+          +-----------------+         +----------------+
                                           |  Trigger
                                           v
                                    +---------------+
                                    | Azure Function|
                                    |      (B)      |
                                    +-------+-------+
                                            |
                   +------------------------+-----------------------------+
                   |                                                      |
          Valid Data to Cosmos DB                               Invalid → ADLS/quarantine
                   |                                              Metadata → ADLS/metadata
                   v
        +----------------------+  
        |   Cosmos DB (ODS)    |
        +----------+-----------+
                   |
                   v
    +----------------------------------+
    |     Databricks Lakehouse        |
    |  Bronze → Silver → Gold Layers  |
    +------------------+---------------+
                       |
                       v
              +-----------------+
              | SQL Data Ware-  |
              | house (Serving) |
              +-------+---------+
                      |
                      v
              +----------------+
              |   Power BI     |
              | Dashboards     |
              +----------------+
```

---

# 3. Azure Resources Used

| Service                              | Purpose                                                                 |
| ------------------------------------ | ----------------------------------------------------------------------- |
| **ADLS Gen2**                        | Raw, Quarantine, Metadata, Bronze, Silver, Gold zones                   |
| **Event Grid**                       | Detects new file uploads in ADLS                                        |
| **Azure Function A**                 | Receives EventGrid events and pushes file metadata into Service Bus     |
| **Service Bus Queue**                | Reliable buffer for downstream processing                               |
| **Azure Function B (Queue Trigger)** | Validates payload → writes valid records to Cosmos DB / invalid to ADLS |
| **Cosmos DB**                        | Operational store for ATM, UPI, and Customer data                       |
| **Databricks**                       | Orchestrates Bronze → Silver → Gold ETL                                 |
| **Azure SQL DB / SQL DWH**           | Serving layer for BI tools                                              |
| **Power BI**                         | Analytics dashboards                                                    |
| **CI/CD (GitHub Actions)**           | Notebook sync + Function App deployment                                 |

---

# 4. ADLS Folder Structure

```
/raw
    /atm
    /upi
    /customers
    /kyc

/quarantine
/metadata

/etl
    /bronze
    /silver
    /gold
```

---

# 5. Ingestion Workflow

### **Step 1 — File lands in ADLS (raw)**

ATM, UPI, Customer, KYC files are uploaded into `/raw`.

### **Step 2 — Event Grid triggers Function A**

Detects `BlobCreated` events and extracts:

* file_url
* file_name
* source_type (ATM/UPI/CUSTOMER/KYC)

### **Step 3 — Function A → Event Grid Trigger**

Minimal enrichment and routing.

### **Step 4 — Function B (Queue-trigger)**

Validates schema, datatype, required fields.

* **If valid:**
  → Insert into Cosmos DB
  → Store metadata in `/metadata`

* **If invalid:**
  → Store full payload in `/quarantine`

### **Step 5 — Databricks Ingestion**

* Reads **Cosmos DB** for transactions & customer profiles
* Reads **KYC directly from ADLS**

---

# 6. Databricks ETL (Lakehouse)

## **6.1 Bronze Layer**

* Raw ingestion from Cosmos DB
* KYC raw ingestion from ADLS
* Light parsing
* No transformations
* Stored as Delta tables

## **6.2 Silver Layer**

* Standardized schema
* Data quality cleanup
* Normalized structures
* Reference cleanup
* Joins across sources
* Error-handled and type-safe

## **6.3 Gold Layer (Final Analytics Layer)**

### **Dimensions**

* `dim_customer`
* `dim_account`
* `dim_kyc`
* `dim_date`

### **SCD-2 Dimensions**

* `dim_customer_scd2`
* `dim_account_scd2`

### **Facts**

* `fact_transactions`
* `fact_customer_profile`

### **Aggregates**

* Daily transaction summary
* Customer monthly spending
* Branch performance
* Channel-wise distribution

---

# 7. Serving Layer (SQL Data Warehouse)

Gold tables and aggregates are written to **Azure SQL DB / SQL Warehouse**, enabling optimized BI reporting.

---

# 8. Power BI Dashboards

Dashboards supported:

* **Customer 360**
* **Fraud Monitoring**
* **Branch Performance**
* **ATM & UPI Analytics**
* **Compliance Reporting**

### Example placeholders:

```
![Customer360](images/powerbi-customer360.png)
![FraudOps](images/powerbi-fraud-ops.png)
![BranchPerformance](images/powerbi-branch-performance.png)
```

---

# 9. CI/CD

This project uses a **lightweight CI/CD workflow** to automate deployments.

### **CI Pipeline**

* Linting & quality checks
* Validate Python Function code
* Validate Databricks notebooks
* Package Function App

### **CD Pipeline**

* Deploy Azure Function App
* Sync Databricks notebooks (via Databricks CLI)
* Optionally trigger Databricks job
* Validate Cosmos DB + ADLS connectivity

Pipeline can be implemented in:

* GitHub Actions
* Azure DevOps Pipelines

(Your repository already includes standard folder structures for this.)

---

# 10. Repository Structure

```
/src
    /function_app
    /notebooks
         /bronze
         /silver
         /gold
    /powerbi
    /sql
/docs
    /images
    architecture.md
/readme.md
```

---

# 11. How to Run the Project

### **Prerequisites**

* Azure subscription
* Cosmos DB + ADLS + Function App + Service Bus
* Databricks Workspace
* SQL Database

### **Execution Steps**

1. Upload sample data into `/raw` in ADLS
2. Event Grid → Functions → Service Bus → Cosmos DB automatically ingests
3. Databricks Jobs run Bronze → Silver → Gold pipeline
4. Gold tables are published to SQL DWH
5. Power BI connects to SQL DWH for reporting
6. CI/CD handles notebook + function deployments

---

# 12. Screenshot Placeholders

Add the following in `/docs/images` or `/images`:

```
![ADLS Structure](https://github.com/user-attachments/assets/2fd56804-5f18-44f6-a41d-a2aaa328718e?raw=true)
![Event Grid Setup](images/event-grid.png)
![Function App Logs](images/function-app-logs.png)
![Cosmos DB Data](images/cosmos-collection.png)
![Databricks Bronze](images/bronze-notebook.png)
![Databricks Gold](images/gold-notebook.png)
![SQL Serving](images/sql-warehouse.png)
![Power BI](images/powerbi.png)
```

---

# 13. Future Enhancements

* Full orchestration using ADF or Synapse
* Real-time analytics using Azure Stream Analytics
* ML-based fraud scoring model
* Expanded SCD2 coverage

---

