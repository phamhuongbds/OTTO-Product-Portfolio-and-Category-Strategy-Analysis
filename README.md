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
