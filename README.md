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

### 4.3  Create dimensional tables
Dimension tables are created for brand, retailer, status, and category to store reusable descriptive attributes separately from the fact table, making the model cleaner, easier to query, and more suitable for future analysis.
~~~sql
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.otto_brand` AS
SELECT DISTINCT brand
FROM `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products`
WHERE brand IS NOT NULL;
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.otto_retailer` AS
SELECT DISTINCT retailer
FROM `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products`
WHERE retailer IS NOT NULL;
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.otto_status` AS
SELECT DISTINCT status
FROM `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products`
WHERE status IS NOT NULL;
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.otto_category` AS
SELECT DISTINCT
  breadcrumbs
FROM `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products`
WHERE breadcrumbs IS NOT NULL
ORDER BY breadcrumbs;
~~~
### 4.4  Create the fact table and the category levels for analysis
This step builds the final fact table and extracts category levels from the breadcrumb path, creating a structured table for analysis.
~~~sql
CREATE OR REPLACE TABLE `otto-ecommerce-analysis.otto_dataset_project.fact_products` AS
SELECT uniq_id,  pid,  product_title, brand, retailer, status, limited,  price, formatted_price, ean, availability, breadcrumbs,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(0)] AS category_level_1,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(1)] AS category_level_2,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(2)] AS category_level_3,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(3)] AS category_level_4,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(4)] AS category_level_5,
  SPLIT(TRIM(breadcrumbs), '|')[SAFE_OFFSET(5)] AS category_level_6,
  scraped_at
FROM `otto-ecommerce-analysis.otto_dataset_project.clean_otto_products`;
~~~
Query Result:
The final fact table combines the cleaned OTTO product data with extracted category levels to create a structured dataset for analysis. Compared with the raw dataset, it removes unnecessary noise and organizes the remaining fields into a clear format that is easier to query and use for business reporting.

| Column Name | Description |
|------------|-------------|
| uniq_id | Unique identifier assigned to each scraped record. |
| pid | Unique product identifier assigned by the website. |
| product_title | Name of the product listed on the website. |
| brand | Brand or manufacturer of the product. |
| retailer | Name of the retailer offering the product. |
| status | Current status of the product (e.g., active, discontinued, sold out). |
| limited | Indicates whether the product is a limited-edition item. |
| price | Raw numerical product price value. |
| formatted_price | Product price displayed in the website's formatted currency representation. |
| ean| European Article Number (EAN) associated with the product. |
| availability | Product stock or availability status at the time of data collection. |
| breadcrumbs | Website navigation path indicating the product's category hierarchy. |
| category_level_1 | Main department or top-level product category. |
| category_level_2 | Secondary category within the main department. |
| category_level_3 | More specific product group within the category hierarchy. |
| category_level_4 | Subcategory or product family classification. |
| category_level_5 | Narrower product segment within the hierarchy. |
| category_level_6 | Most detailed category label available from the breadcrumb path. |
| scraped_at | Timestamp indicating when the product information was collected from the website. |

## 5. Business questions
### 5.1 Which brands have the largest assortment and what is their pricing position?
~~~sql
WITH brand_summary AS 
  (SELECT brand,
    COUNT(*) AS product_count,
    ROUND(AVG(price),2) AS avg_price
  FROM `otto-ecommerce-analysis.otto_dataset_project.fact_products`
  WHERE brand IS NOT NULL
    AND price IS NOT NULL
  GROUP BY brand)
SELECT *
FROM brand_summary
ORDER BY product_count DESC, avg_price DESC;
~~~
Query Result: 
| Brand | Product Count | Avg Price (€) |
|---------|-------------:|-------------:|
| K-S-Trade | 90 | 16.80 |
| Abakuhaus | 83 | 24.08 |
| CALVENDO | 55 | 27.17 |
| Reyher | 50 | 399.82 |
| DeinDesign | 49 | 23.28 |
| SONSTIGE | 35 | 195.16 |
| atFoliX | 23 | 9.07 |
| Posterlounge | 20 | 10.50 |
| vhbw | 19 | 28.76 |
| vidaXL | 18 | 114.69 |

*Full result table refers to file Result 5.1*

The result shows that the brands with the largest assortments are K-S-Trade, Abakuhaus, CALVENDO, Reyher, and DeinDesign. Their pricing positions are very different: some are value-oriented, like K-S-Trade and atFoliX, while others are premium positioned, especially Reyher and SONSTIGE, which have much higher average prices.

### 5.2 Which retailers offer the largest and most diverse product portfolios?
~~~sql
WITH retailer_portfolio AS 
    (SELECT
        retailer,
        COUNT(*) AS product_count,
        COUNT(DISTINCT brand) AS brand_count,
        ROUND(AVG(price), 2) AS avg_price,
        COUNTIF(status = 'soldout') AS sold_out_products,
        ROUND(COUNTIF(status = 'soldout') * 100.0 / COUNT(*),2) AS sold_out_rate
    FROM `otto-ecommerce-analysis.otto_dataset_project.fact_products`
    WHERE retailer IS NOT NULL
    GROUP BY retailer)
SELECT
    retailer,
    product_count,
    brand_count,
    avg_price,
    sold_out_products,
    sold_out_rate
FROM retailer_portfolio
ORDER BY product_count DESC, brand_count DESC;
~~~
Query Result:
| Retailer | Product Count | Brand Count | Avg Price (€) | Sold-Out Products | Sold-Out Rate (%) |
|----------|-------------:|------------:|--------------:|------------------:|------------------:|
| OTTO | 426 | 280 | 337.51 | 107 | 25.12 |
| Löchel Industriebedarf | 108 | 15 | 256.01 | 12 | 11.11 |
| K. S. company GmbH | 90 | 1 | 16.80 | 6 | 6.67 |
| Color-D Textile GmbH | 83 | 1 | 24.08 | 2 | 2.41 |
| Home & Play | 55 | 1 | 27.17 | 0 | 0.00 |
| DeinDesign | 49 | 1 | 23.28 | 0 | 0.00 |
| schutzfolien24 | 38 | 3 | 14.84 | 0 | 0.00 |
| mirapodo #ft5_slash# myToys | 34 | 30 | 43.14 | 1 | 2.94 |
| atFoliX | 30 | 2 | 8.42 | 0 | 0.00 |
| WE LOVE BAGS | 28 | 25 | 123.37 | 23 | 82.14 |

**Full result table refers to file Result 5.2*

