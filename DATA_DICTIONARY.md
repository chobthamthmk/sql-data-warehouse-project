# Data Dictionary

## Layered architecture

| Layer | Purpose | Main objects |
|---|---|---|
| **Source** | Raw CRM and ERP CSV exports. | `datasets/source_crm`, `datasets/source_erp` |
| **Bronze** | Raw ingestion with source-aligned columns and minimal transformation. | `bronze.crm_*`, `bronze.erp_*` |
| **Silver** | Cleaned, standardized, typed, and enriched data. | `silver.crm_*`, `silver.erp_*` |
| **Gold** | Business-ready star schema for reporting and BI. | `gold.dim_customers`, `gold.dim_products`, `gold.fact_sales` |

## Source layer

| Source | Files | Loaded into |
|---|---|---|
| CRM | `cust_info.csv`, `prd_info.csv`, `sales_details.csv` | `bronze.crm_cust_info`, `bronze.crm_prd_info`, `bronze.crm_sales_details` |
| ERP | `CUST_AZ12.csv`, `LOC_A101.csv`, `PX_CAT_G1V2.csv` | `bronze.erp_cust_az12`, `bronze.erp_loc_a101`, `bronze.erp_px_cat_g1v2` |

## Bronze layer

The Bronze layer preserves source structures for traceability and repeatable reloads.

| Table | Grain | Main columns |
|---|---|---|
| `bronze.crm_cust_info` | One row per CRM customer record | `cst_id`, `cst_key`, `cst_firstname`, `cst_lastname`, `cst_marital_status`, `cst_gndr`, `cst_create_date` |
| `bronze.crm_prd_info` | One row per CRM product record/version | `prd_id`, `prd_key`, `prd_nm`, `prd_cost`, `prd_line`, `prd_start_dt`, `prd_end_dt` |
| `bronze.crm_sales_details` | One row per sales order line | `sls_ord_num`, `sls_prd_key`, `sls_cust_id`, `sls_order_dt`, `sls_ship_dt`, `sls_due_dt`, `sls_sales`, `sls_quantity`, `sls_price` |
| `bronze.erp_cust_az12` | One row per ERP customer record | `cid`, `bdate`, `gen` |
| `bronze.erp_loc_a101` | One row per ERP customer-location record | `cid`, `cntry` |
| `bronze.erp_px_cat_g1v2` | One row per ERP product-category record | `id`, `cat`, `subcat`, `maintenance` |

## Silver layer

The Silver layer keeps the source naming where useful, adds `dwh_create_date`, and stores cleaned values used by the Gold views.

| Table | Grain | Transformations / important columns |
|---|---|---|
| `silver.crm_cust_info` | One row per latest CRM customer record | Standardized customer text and demographics; `dwh_create_date` audit timestamp. |
| `silver.crm_prd_info` | One row per product version | Derived `cat_id`; typed `prd_start_dt` and `prd_end_dt`; `dwh_create_date`. |
| `silver.crm_sales_details` | One row per sales order line | Converts integer dates to `DATE`; retains sales, quantity, and price measures. |
| `silver.erp_cust_az12` | One row per ERP customer record | Standardized customer key, birthdate, and gender; `dwh_create_date`. |
| `silver.erp_loc_a101` | One row per customer-location record | Standardized country values; `dwh_create_date`. |
| `silver.erp_px_cat_g1v2` | One row per product-category record | Standardized category, subcategory, and maintenance values; `dwh_create_date`. |

## Gold layer

## Overview
The Gold Layer is the business-level data representation, structured to support analytical and reporting use cases. It consists of **dimension tables** and **fact tables** for specific business metrics.

---

### 1. **gold.dim_customers**
- **Purpose:** Stores customer details enriched with demographic and geographic data.
- **Columns:**

| Column Name      | Data Type     | Description                                                                                   |
|------------------|---------------|-----------------------------------------------------------------------------------------------|
| customer_key     | INT           | Surrogate key uniquely identifying each customer record in the dimension table.               |
| customer_id      | INT           | Unique numerical identifier assigned to each customer.                                        |
| customer_number  | NVARCHAR(50)  | Alphanumeric identifier representing the customer, used for tracking and referencing.         |
| first_name       | NVARCHAR(50)  | The customer's first name, as recorded in the system.                                         |
| last_name        | NVARCHAR(50)  | The customer's last name or family name.                                                     |
| country          | NVARCHAR(50)  | The country of residence for the customer (e.g., 'Australia').                               |
| marital_status   | NVARCHAR(50)  | The marital status of the customer (e.g., 'Married', 'Single').                              |
| gender           | NVARCHAR(50)  | The gender of the customer (e.g., 'Male', 'Female', 'n/a').                                  |
| birthdate        | DATE          | The date of birth of the customer, formatted as YYYY-MM-DD (e.g., 1971-10-06).               |
| create_date      | DATE          | The date and time when the customer record was created in the system|

---

### 2. **gold.dim_products**
- **Purpose:** Provides information about the products and their attributes.
- **Columns:**

| Column Name         | Data Type     | Description                                                                                   |
|---------------------|---------------|-----------------------------------------------------------------------------------------------|
| product_key         | INT           | Surrogate key uniquely identifying each product record in the product dimension table.         |
| product_id          | INT           | A unique identifier assigned to the product for internal tracking and referencing.            |
| product_number      | NVARCHAR(50)  | A structured alphanumeric code representing the product, often used for categorization or inventory. |
| product_name        | NVARCHAR(50)  | Descriptive name of the product, including key details such as type, color, and size.         |
| category_id         | NVARCHAR(50)  | A unique identifier for the product's category, linking to its high-level classification.     |
| category            | NVARCHAR(50)  | The broader classification of the product (e.g., Bikes, Components) to group related items.  |
| subcategory         | NVARCHAR(50)  | A more detailed classification of the product within the category, such as product type.      |
| maintenance_required| NVARCHAR(50)  | Indicates whether the product requires maintenance (e.g., 'Yes', 'No').                       |
| cost                | INT           | The cost or base price of the product, measured in monetary units.                            |
| product_line        | NVARCHAR(50)  | The specific product line or series to which the product belongs (e.g., Road, Mountain).      |
| start_date          | DATE          | The date when the product became available for sale or use, stored in|

---

### 3. **gold.fact_sales**
- **Purpose:** Stores transactional sales data for analytical purposes.
- **Columns:**

| Column Name     | Data Type     | Description                                                                                   |
|-----------------|---------------|-----------------------------------------------------------------------------------------------|
| order_number    | NVARCHAR(50)  | A unique alphanumeric identifier for each sales order (e.g., 'SO54496').                      |
| product_key     | INT           | Surrogate key linking the order to the product dimension table.                               |
| customer_key    | INT           | Surrogate key linking the order to the customer dimension table.                              |
| order_date      | DATE          | The date when the order was placed.                                                           |
| shipping_date   | DATE          | The date when the order was shipped to the customer.                                          |
| due_date        | DATE          | The date when the order payment was due.                                                      |
| sales_amount    | INT           | The total monetary value of the sale for the line item, in whole currency units (e.g., 25).   |
| quantity        | INT           | The number of units of the product ordered for the line item (e.g., 1).                       |
| price           | INT           | The price per unit of the product for the line item, in whole currency units (e.g., 25).      |
