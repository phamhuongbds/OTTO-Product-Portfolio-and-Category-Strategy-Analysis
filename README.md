# OTTO-Product-Portfolio-and-Category-Strategy-Analysis
Using SQL to analyze the OTTO product data.
## 1. Project Introduction

### 1.1 Business Context
OTTO's merchandising team wants to optimize its product portfolio by identifying:
- High-priority categories for investment.
- Strategic brand partnerships.
- Premium product opportunities.
- Inventory and availability risks.
- Product segments with the greatest growth potential.

The goal is to support merchandising, category management, and marketing investment decisions using the current product catalog.

### 1.2 Business Goal
Determine which categories, brands, and retailer partnerships should receive increased business investment based on assortment size, pricing position, and product availability.

## 2. Data Introduction

Source: [Crawlfeeds OTTO E-Commerce Products Sample Database](https://crawlfeeds.com/datasets/otto-ecommerce-products-sample-database)

The OTTO raw dataset is collected from the Crawlfeeds website and contains product-level information across retailers, brands, pricing, and availability.

| Column Name | Description |
|---|---|
| URL | Webpage URL where the product information was collected. |
| PID | Unique product identifier assigned by the website. |
| Product Title | Name of the product listed on the website. |
| Brand | Brand or manufacturer of the product. |
| Status | Current status of the product, such as active or discontinued. |
| Limited | Indicates whether the product is a limited edition item. |
| Retailer | Name of the retailer offering the product. |
| Formatted Price | Product price displayed in the website's formatted currency representation. |
| EAN | European Article Number associated with the product. |
| MOIN | Additional product identifier used by the retailer or website. |
| Price | Raw numerical product price value. |
| Description | Product description provided on the website. |
| Details | Additional product specifications and details. |
| Images | URLs or references to product images. |
| Availability | Product stock or availability status. |
| Breadcrumbs | Website navigation path showing the product's category hierarchy. |
| Uniq ID | Unique identifier assigned to each scraped record. |
| Scraped At | Timestamp indicating when the data was collected from the website. |

## 3. Analytic Approach

The analysis follows these steps:

1. Load the raw data into BigQuery.
2. Clean and standardize the fields in a staging table.
3. Remove duplicates using `ROW_NUMBER()` with `QUALIFY`.
4. Split breadcrumbs into category levels.
5. Create dimension tables.
6. Build a fact table as the main table for analysis.
7. Use SQL queries to answer the business questions.

## 4. Data cleaning and processing
### 4.1 Staging table
The raw OTTO dataset is loaded into a staging table to clean text fields, convert data types, and prepare the data for deduplication and modeling.
~~~sql
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.stg_otto_products` AS
SELECT
  TRIM(`Product title`) AS product_title,
  TRIM(`Brand`) AS brand,
  LOWER(TRIM(`Status`)) AS status,
  CAST(`Limited` AS BOOL) AS limited,
  TRIM(`Retailer`) AS retailer,
  SAFE_CAST(`Formatted price` AS FLOAT64) AS formatted_price,
  SAFE_CAST(`Ean` AS INT64) AS ean,
  TRIM(`Moin`) AS moin,
  SAFE_CAST(`Price` AS FLOAT64) AS price,
  TRIM(`Description`) AS description,
  TRIM(`Details`) AS details,
  TRIM(`Images`) AS images,
  TRIM(`Availability`) AS availability,
  TRIM(`Breadcrumbs`) AS breadcrumbs,
  TRIM(`Uniq id`) AS uniq_id,
  `Scraped at` AS scraped_at,
  TRIM(`Url`) AS url,
  TRIM(`Pid`) AS pid
FROM `otto-ecommerce-analysis.otto_dataset_project.Raw_otto_dataset`
WHERE `Product title` IS NOT NULL;
~~~
Query Result:
The staging table trims text fields, converts capital letters to lowercase, and standardizes the raw data for cleaning and analysis.

### 4.2 Deduplication
This step removes duplicate records by keeping the most recent entry for each product based on product ID, unique ID, title, brand, and retailer.
~~~sql
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products` AS
SELECT *
FROM `otto-ecommerce-analysis.otto_dataset_project.stg_otto_products`
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY pid, uniq_id, product_title, brand, retailer
  ORDER BY scraped_at DESC
) = 1;
~~~
Query Result:
After deduplication, the table still contained 2,223 rows, identical to the original row count, which suggests the raw dataset was already mostly unique at this level of grouping.

### 4.3 Dimension tables
