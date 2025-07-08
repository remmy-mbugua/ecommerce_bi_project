# Data Limitations & Challenges

This document outlines the key challenges, assumptions, and limitations encountered during the data preparation and analysis phase of the project.

## 1. Missing Customer IDs
Approximately 25% of transactions lacked a `CustomerID`. These rows were excluded to maintain the accuracy of customer-level analysis such as retention, RFM segmentation, and cohort tracking.

## 2. No Product Hierarchy or Categories
The dataset did not include product categories or hierarchies. A manual classification system was implemented by mapping top-selling products to categories based on name patterns and business logic. This covered ~80% of revenue.

## 3. Returns Without Matching Sales
The dataset spans only one year, meaning some return transactions had no corresponding sales within the available window. These returns were excluded if the customer's total returns exceeded their known purchases for that product.

## 4. Limited Time Frame
Data covers a single year (Dec 2010 – Dec 2011), limiting the ability to conduct long-term analyses such as lifetime value, seasonality trends, or year-over-year comparisons.

## 5. Regional Skew
Roughly 80% of transactions come from the UK. While the model supports regional analysis, insights may be biased toward UK market dynamics and may not generalise well to other countries.

## 6. No Order Status or Shipping Information
There is no visibility into cancelled orders, fulfillment times, or delivery success. This limits the scope of operational analysis.

## 7. No Cost or Margin Data
Only revenue (via `UnitPrice`) is provided. Without cost of goods sold (COGS), true profitability cannot be assessed. Profit-oriented KPIs (e.g. margin per unit) are based on assumptions or excluded.

---

These limitations were addressed or mitigated where possible and clearly noted in documentation, reports, and business insights. They reflect common real-world data issues and demonstrate the need for practical, defensible assumptions in BI work.
