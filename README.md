# retail-Sales-data-warehouse-iics
End-to-End Retail Sales Data Warehouse using IICS
# Retail Sales Data Warehouse Project (IICS)

## 📌 Project Overview
This project demonstrates an end-to-end retail Sales data warehouse built using
Informatica Intelligent Cloud Services (IICS) and SQL Server.

The goal is to design and implement a scalable dimensional model,
handle incremental loads, and ensure data quality through validations.

---

## 🏗️ Architecture
- Source: CSV files
- Staging Layer: stg schema
- Data Warehouse: dw schema
- Facts: fact schema
- ETL Tool: Informatica IICS
- Database: SQL Server

---

## 📊 Data Model
### Dimensions
- dim_date
- dim_customer (SCD Type 2)
- dim_product (SCD Type 1)
- dim_store (SCD Type 1)

### Facts
- fact_orders (Order-level grain)
- fact_order_items (Product-level grain)

---

## 🧱 Fact Grain & Modeling Decisions

### fact_orders
- Grain: **One row per order**
- Each record represents a completed customer order
- Measures:
  - total_amount
- Dimensions linked:
  - dim_date
  - dim_customer
  - dim_store

### fact_order_items
- Grain: **One row per product per order**
- An order with multiple products generates multiple rows
- Measures:
  - quantity
  - unit_price
  - line_total
- Dimensions linked:
  - dim_date
  - dim_customer
  - dim_product
  - dim_store

### Why two fact tables?
Separating order-level and item-level data avoids:
- Data duplication
- Incorrect aggregations
- Performance issues

This design supports both:
- High-level sales analysis (fact_orders)
- Detailed product analysis (fact_order_items)


---

## 🔄 ETL Design
- Separate staging, dimension, and fact loads
- Surrogate keys for all dimensions
- Business keys used for lookups
- Null-safe change detection logic
- Insert and update paths clearly separated


## 🔁 Taskflow Orchestration & Load Order

This project follows a **controlled, dependency-driven taskflow design**
to ensure data consistency and referential integrity.

### 🥉 Stage Layer Load (Parallel)
All staging tables are loaded in parallel since they are independent.

- stg.stg_customers
- stg.stg_products
- stg.stg_stores
- stg.stg_orders
- stg.stg_order_items

✔ Purpose:
- Raw data ingestion
- No dependencies
- Fast and scalable

---

### 🥈 Dimension Layer Load (Sequential)
Dimension tables are loaded **after staging** and follow dependency order.

1. **dim_date**
   - Loaded first (static dimension)

2. **dim_customer**
   - SCD Type 2 logic
   - Depends on `stg_customers`

3. **dim_product**
   - SCD Type 1 logic
   - Depends on `stg_products`

4. **dim_store**
   - SCD Type 1 logic
   - Depends on `stg_stores`

✔ Purpose:
- Generate surrogate keys
- Handle slowly changing dimensions
- Prepare lookup-ready dimensions

---

### 🥇 Fact Layer Load (Sequential)
Fact tables are loaded **only after all dimensions are successfully loaded**.

1. **fact_orders**
   - Grain: One row per order
   - Depends on:
     - dim_date
     - dim_customer
     - dim_store

2. **fact_order_items**
   - Grain: One row per product per order
   - Depends on:
     - dim_date
     - dim_customer
     - dim_product
     - dim_store

✔ Purpose:
- Maintain referential integrity
- Prevent orphan foreign keys
- Ensure accurate aggregations

---

## 🚦 Error Handling & Restartability

- Each layer runs in its own taskflow
- Failure in one layer prevents downstream execution
- Taskflows can be restarted from the failed step
- Ensures production-grade reliability

---

## ✅ Why This Orchestration Strategy?

- Prevents partial data loads
- Guarantees dimension availability before facts
- Supports incremental and full loads
- Aligns with enterprise ETL best practices



## 🔄 Slowly Changing Dimension (SCD) Strategy

This project implements different Slowly Changing Dimension (SCD) strategies
based on business requirements and data volatility.

### 🧍 dim_customer — SCD Type 2 (History Preserved)
Customer attributes can change over time (address, phone, email).
To preserve historical accuracy:

- A new row is inserted when a change is detected
- Previous record is expired using `end_dt`
- `is_current` flag identifies the active record
- Surrogate key changes for each version

**Change Detection Logic**
- Null-safe comparison using default placeholders
- Only non-key attributes are compared
- Business key (`customer_id`) is never updated

**Why SCD Type 2?**
Historical customer attributes must be retained to ensure:
- Correct past sales reporting
- Accurate customer behavior analysis

---

### 📦 dim_product — SCD Type 1 (Overwrite Changes)
Product attributes are corrected or updated without historical tracking.

- Existing records are updated in place
- Surrogate key remains unchanged
- No history maintained

**Change Detection Logic**
- Null-safe comparison for all descriptive attributes
- `product_id` is treated as immutable business key
- Surrogate key is never updated

**Why SCD Type 1?**
Product changes do not require historical tracking
and correcting data is preferred over retaining old values.

---

### 🏬 dim_store — SCD Type 1 (Overwrite Changes)
Store location details may change due to data corrections.

- Updates overwrite existing values
- No historical versions created
- Surrogate key remains constant

---

### 📅 dim_date — Static Dimension
The date dimension is pre-generated and does not change.

- Loaded once
- No updates required
- Used for all time-based analysis

---

## 🔐 Surrogate Key Handling Rules

- Surrogate keys are system-generated
- Never updated once assigned
- Used only for fact table relationships
- Business keys are used for lookups only

---

## 🧠 Design Principles Applied

- Business keys identify records
- Surrogate keys join facts to dimensions
- Change detection excludes business keys
- Audit columns managed at target level
- Insert and update logic clearly separated


## ♻️ Incremental Load Strategy & Audit Columns

This project implements a **batch-based incremental loading strategy**
to efficiently process only new and changed records.

---

### 🧾 Batch Control

- Each load is driven by a `batch_id`
- Batch ID format: `YYYYMMDD_HHMMSS`
- Used across:
  - Staging
  - Dimensions
  - Facts

✔ Enables traceability and reprocessing

---

### 🕒 Audit Columns Used

All tables include standard audit columns:

- `insert_dt` — record creation timestamp
- `update_dt` — last update timestamp
- `load_user` — ETL execution user
- `batch_id` — identifies the load batch
- `file_name` — source file name
- `file_row_number` — row position in source file

---

### 🔄 Staging Layer Incremental Logic

- Data loaded per file per batch
- Duplicate file detection prevents accidental reprocessing of the same file.
- File-level validation before processing

✔ Ensures raw data consistency

---

### 🧱 Dimension Incremental Logic

- Business keys used for lookups
- New records → **INSERT**
- Changed records → **UPDATE (SCD logic)**
- Unchanged records → **IGNORED**

✔ Prevents unnecessary updates

---

### 📊 Fact Incremental Logic

- Facts loaded only for current batch
- Foreign keys resolved via dimension lookups
- Records are rejected or logged if corresponding dimension keys are missing.

✔ Guarantees referential integrity

---

## 🎯 Benefits of This Strategy

- Scalable for large data volumes
- Easy restart and rollback
- Clear data lineage
- Production-ready design


---

## ✅ Data Validation
- Row count reconciliation
- Null checks
- Duplicate checks
- Surrogate key validation
- Fact-to-dimension integrity checks

---

## 🚀 Skills Demonstrated
- Dimensional Modeling
- SCD Type 1 & Type 2
- Fact grain design
- IICS mappings & taskflows
- SQL validations
- Production-ready ETL design

---


## 📈 How This Project Is Used
This data warehouse enables business users and analysts to:
- Track daily and monthly sales trends
- Identify top-performing products and stores
- Analyze customer lifetime value and repeat purchases
- Ensure accurate reporting using validated, audit-ready data

All analytical queries are built on well-defined fact and dimension tables,
following industry-standard dimensional modeling practices.

---

## 👤 Author
Built and documented by Raghukumar Kasthuri
