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

The table shows that OTTO is the dominant retailer, offering 426 products across 280 brands, significantly exceeding all other retailers in both assortment size and brand diversity. After OTTO, a small number of retailers maintain relatively large product portfolios, including Löchel Industriebedarf (108 products), K. S. company GmbH (90 products), and Color-D Textile GmbH (83 products). However, most retailers offer only a limited assortment, indicating that the marketplace relies heavily on a few key retail partners.

### 5.3 Which categories have the highest sold-out rate?
~~~sql
SELECT
  category_level_2,
  COUNT(*) AS total_products,
  COUNTIF(status = 'soldout') AS sold_out_products,
  ROUND(SAFE_DIVIDE(COUNTIF(status = 'soldout'), COUNT(*)),2) AS sold_out_rate
FROM `otto-ecommerce-analysis.otto_dataset_project.fact_products`
WHERE category_level_2 IS NOT NULL
GROUP BY category_level_2
ORDER BY sold_out_rate DESC, total_products DESC;
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
| mirapodo / myToys | 34 | 30 | 43.14 | 1 | 2.94 |
| atFoliX | 30 | 2 | 8.42 | 0 | 0.00 |
| WE LOVE BAGS | 28 | 25 | 123.37 | 23 | 82.14 |

**Full result table refers to file Result 5.3*

The table shows that a few categories, such as Kommunikation and Medien, show a 100% sold-out rate; however, these results are based on only one product and are not representative of overall demand. Among categories with meaningful product volumes, Damen records the highest number of sold-out products (88), followed by Herren (30) and Schuhe (22), indicating strong demand or potential inventory shortages. The relatively high sold-out rates in Damen (34%), Schuhe (30%), and Herren (27%) suggest that these categories may require closer inventory monitoring. 

### 5.4 Do limited products have different pricing?
~~~sql
SELECT 
  limited,
  COUNT(*) AS product_count,
  ROUND(AVG(price), 2) AS avg_price,
  ROUND(MIN(price), 2) AS min_price,
  ROUND(MAX(price), 2) AS max_price
FROM `otto-ecommerce-analysis.otto_dataset_project.fact_products`
WHERE price IS NOT NULL
GROUP BY limited
ORDER BY limited;
~~~
Query Result 
| limited | Product Count | Avg Price (€) | Min Price (€) | Max Price (€) |
|--------|-------------:|---------------:|--------------:|--------------:|
| FALSE | 1701 | 142.58 | 1.09 | 22236.37|
| TRUE | 522 | 189.45 | 5.20 | 14748.99 |

The results show that limited products are priced higher on average than non-limited products. Limited items have an average price of €189.45, compared with €142.58 for regular products, and they also have a much higher maximum price, which suggests they are positioned as more premium or exclusive items.

### 5.5 Which brands dominate within each retailer?
~~~sql
WITH brand_retailer AS 
  (SELECT f.retailer, f.brand, COUNT(*) AS product_count
  FROM `otto-ecommerce-analysis.otto_dataset_project.fact_products` f
  LEFT JOIN `otto-ecommerce-analysis.otto_dataset_project.otto_retailer` r
    ON f.retailer = r.retailer
  WHERE f.retailer IS NOT NULL
    AND f.brand IS NOT NULL
  GROUP BY f.retailer, f.brand), ranked AS 
  (SELECT *,  RANK() OVER ( PARTITION BY retailer
      ORDER BY product_count DESC) AS brand_rank
  FROM brand_retailer)
SELECT *
FROM ranked
WHERE brand_rank <= 5
ORDER BY retailer, brand_rank;
~~~
Query Result:
| Retailer | Brand | Product Count | Brand Rank |
|---|---|---:|---:|
| 123moebel | White Label Living | 1 | 1 |
| 1A PHOTO PORST | 1A PHOTO PORST | 1 | 1 |
| 1a-Handelsagentur | euro3plast | 1 | 1 |
| 2JB | ZERO G | 2 | 1 |
| 3DEAL | Kickers | 1 | 1 |
| 440s - for fourties | AM Design | 1 | 1 |
| 440s - for fourties | 440s | 1 | 1 |
| 440s - for fourties | Mars & More | 1 | 1 |
| 4big.fun | Cheffinger | 1 | 1 |
| 58 auf'm Kessel | 58 aufm Kessel | 1 | 1 |

**Full result table refers to file Result 5.5*

The results show that each retailer tends to have one or a few dominant brands, so assortment is often highly concentrated rather than evenly distributed. Examples include K. S. company GmbH with K-S-Trade (90 products), Color-D Textile GmbH with Abakuhaus (83), Home & Play with CALVENDO (55), and Löchel Industriebedarf with Reyher (50). This suggests these retailers are closely tied to a few core brands rather than operating as broad multi-brand assortments.

## 6. Business recommendation
- Differentiate brand strategy by price position. Value brands can be expanded to support volume, while premium brands should be emphasized in categories where customers are more willing to pay higher prices. Brands such as Reyher and vidaXL should be reviewed to check whether their premium pricing is supported by strong assortment value and visibility.
- Prioritize key portfolio partners. Retailers such as OTTO, Löchel Industriebedarf, mirapodo / myToys, and heyconnect combine scale with brand depth and are therefore important partners. Retailers with strong concentration, such as K. S. company GmbH, Color-D Textile GmbH, Home & Play, and DeinDesign, should be reviewed to decide whether they should expand their brand mix or remain specialist sellers.
- Address stock risk in large categories. Damen, Schuhe, Herren, Sport, and Kinder have meaningful product counts and noticeable sold-out rates, so these categories should be prioritized for replenishment and assortment planning.
- Position limited products as a premium segment. Limited products appear to carry higher prices, so OTTO can highlight them more in merchandising and search placement. At the same time, OTTO should monitor whether the premium position affects conversion or requires targeted promotions.
- Use brand concentration strategically. Retailers with a small number of dominant brands may be easier to scale when those brands perform well. Retailers with weak concentration or a very narrow mix should be reviewed to determine whether the current setup is too limited or already optimized.
