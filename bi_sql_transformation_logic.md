
# BI SQL Transformation Logic

## Fact Table

Filters raw transaction data to include only revenue-relevant rows.

**Key Steps:**
- Removed non-revenue rows (transactions with `UnitPrice <= 0`).
- Removed transactions without a `CustomerID` to maintain accurate customer segmentation and cohort analyses.
- Classified transactions as either **Sale** or **Return** for downstream logic.
- Cleaned returns to exclude cases where quantity returned exceeds the quantity originally sold for that customer-product.

```sql
WITH raw_sales_fact AS (
SELECT 
    InvoiceNo AS invoice_no,
    CustomerID AS customer_id,
    TRIM(INITCAP(Description)) AS product_name,
    DENSE_RANK() OVER (ORDER BY Description) AS product_id,
    TO_DATE(SPLIT_PART(InvoiceDate, ' ', 1), 'MM/DD/YYYY') AS invoice_date,
    TO_TIMESTAMP(InvoiceDate, 'MM/DD/YYYY HH24:MI')::TIME AS transaction_time,
    CASE WHEN Quantity >= 0 THEN 'Sale' ELSE 'Return' END AS transaction_type,
    Quantity,
    UnitPrice,
    Quantity * UnitPrice AS total_amount
FROM ecommerce
WHERE UnitPrice > 0
AND CustomerID <> ''
),

total_sales AS ( -- aggregates quantity sold by customer-product pair
  SELECT customer_id, product_id, SUM(quantity) AS total_quantity_sold
  FROM raw_sales_fact
  WHERE transaction_type = 'Sale'
  GROUP BY customer_id, product_id
),
total_returns AS ( -- aggregates quantity returned by customer-product pair
  SELECT customer_id, product_id, SUM(ABS(quantity)) AS total_quantity_returned
  FROM raw_sales_fact
  WHERE transaction_type = 'Return'
  GROUP BY customer_id, product_id
),
valid_return_customers AS ( -- provides the customer-product pairs whose quantity returned doesn't exceed the quantity sold
  SELECT s.customer_id, s.product_id
  FROM total_sales s
  LEFT JOIN total_returns r  -- LEFT JOIN to retain all sales, and only valid returns
    ON s.customer_id = r.customer_id AND s.product_id = r.product_id
  WHERE COALESCE(r.total_quantity_returned, 0) <= s.total_quantity_sold 
)

SELECT fs.*
FROM raw_sales_fact fs
LEFT JOIN valid_return_customers vrc
  ON fs.customer_id = vrc.customer_id AND fs.product_id = vrc.product_id
WHERE 
  fs.transaction_type = 'Sale'
  OR (
    fs.transaction_type = 'Return'
    AND vrc.customer_id IS NOT NULL
  )
...
```

## Fact Cohort

Tracks user activity by cohort month and measures engagement across time.

**Key Steps:**
- Determine the cohort month (first purchase month) for each customer.
- Track monthly activity since acquisition.
- Calculate months since acquisition.
- Aggregate the number of active customers each month.

```sql
CREATE TABLE fact_cohort AS
WITH acquisition AS ( -- shows each customer and their acquisition month, by country
SELECT 
	c.country_id,
	f.customer_id, 
	DATE_TRUNC('month', MIN(f.invoice_date))::date AS cohort_month
  FROM dim_customer c
  JOIN fact_sales f ON c.customer_id = f.customer_id
GROUP BY c.country_id, f.customer_id
),

activity AS ( -- Get all purchase activity for each customer, keeping track of their cohort month
SELECT
	a.country_id,
	a.customer_id,
	a.cohort_month,
	DATE_TRUNC('month', f.invoice_date)::date AS activity_month
	   
  FROM acquisition a
  JOIN fact_sales f
	ON a.customer_id = f.customer_id
),

months_since AS ( -- Calculates the number of full months between the cohort month and each activity month
    SELECT 
	country_id,
        cohort_month,
        DATE_PART('month', AGE(activity_month, cohort_month)) +
        (DATE_PART('year', AGE(activity_month, cohort_month)) * 12) AS months_since_acquisition,
        customer_id
    FROM activity
    WHERE activity_month >= cohort_month

SELECT 
    country,
    cohort_month,
    months_since_acquisition,
    COUNT(DISTINCT customer_id) AS active_customers
FROM months_since
GROUP BY country, cohort_month, months_since_acquisition
ORDER BY country, cohort_month, months_since_acquisition;

ALTER TABLE fact_cohort
ADD COLUMN cohort_size INTEGER;

WITH cohort_sizes AS (
SELECT cohort_month, active_customers
  FROM fact_cohort
 WHERE months_since_acquisition = 0
 )
 
UPDATE fact_cohort f
SET cohort_size = c.active_customers
FROM cohort_sizes c
WHERE c.cohort_month = f.cohort_month

...
```

## Fact Customer RFM Monthly

Tracks monthly RFM scores and customer segments over time.

**Key Steps:**
- Create monthly snapshot dates.
- Aggregate customer transactions up to each snapshot date.
- Calculate recency, frequency, and monetary values.
- Score RFM dimensions (1 to 5 scale).
- Derive customer segments and segment groups.
- Track segment migration for up to 4 months.

```sql
CREATE TABLE fact_customer_rfm_monthly AS
WITH months AS ( -- takes the start of each month from dim_date as the rfm snapshot date
SELECT DISTINCT DATE_TRUNC('month', date)::date AS snapshot_date
  FROM dim_date
WHERE date <= (SELECT MAX(invoice_date) FROM fact_sales)
),

rfm_snapshot AS ( -- generates the frequency and monetary stats for each customer, as well as last purchase date 
SELECT f.customer_id,
	   m.snapshot_date,
	   TO_CHAR(m.snapshot_date,'YYYY-MM') AS  month_label,
	   MAX(f.invoice_date) AS last_purchase_date,
	   COUNT(DISTINCT f.invoice_no) AS frequency,
	   SUM(f.total_amount) AS monetary
	 
  FROM months m
  JOIN fact_sales f 
    ON f.invoice_date <= m.snapshot_date -- ensures that stats derive from dates up to and including the snapshot date
GROUP BY f.customer_id, m.snapshot_date
),

recency_calc AS ( -- creates recency stat for each customer
SELECT *, 
	snapshot_date::date - r.last_purchase_date::date AS recency
  FROM rfm_snapshot r
),

scoring AS ( 
SELECT 
	r.*,
	CASE 
	WHEN recency <= 30 THEN 5
	WHEN recency <= 60 THEN 4
	WHEN recency <= 90 THEN 3
	WHEN recency <= 180 THEN 2
	ELSE 1
	END AS recency_score,
	
	CASE 
	WHEN frequency >= 10 THEN 5
	WHEN frequency BETWEEN 6 AND 9 THEN 4
	WHEN frequency BETWEEN 3 AND 5 THEN 3
	WHEN frequency = 2 THEN 2
	ELSE 1 
	END AS frequency_score,

	CASE 
	WHEN monetary >= 685 THEN 5
	WHEN monetary BETWEEN 513 AND 684 THEN 4
	WHEN monetary BETWEEN 342 AND 512 THEN 3
	WHEN monetary BETWEEN 175 AND 341 THEN 2
	ELSE 1 
	END AS monetary_score
	
FROM recency_calc r
),

segments AS ( -- segments customers based on scoring
SELECT *,
	CONCAT(recency_score, frequency_score, monetary_score) AS rfm_code,
	CASE
	WHEN recency_score = 5 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'Champions'
  	WHEN recency_score >= 4 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'Loyal Customers'
  	WHEN recency_score >= 4 AND frequency_score = 2 AND monetary_score >= 2 THEN 'Potential Loyalists'
  	WHEN recency_score >= 4 AND frequency_score = 1 THEN 'New Customers'
  	WHEN monetary_score = 5 AND frequency_score <= 2 AND recency_score >= 3 THEN 'Big Spenders'
  	WHEN recency_score <= 2 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'High Risk'
  	WHEN recency_score <= 2 AND frequency_score <= 2 AND monetary_score <= 2 THEN 'Hibernating'
	WHEN recency_score >= 3 AND frequency_score <= 2 AND monetary_score <= 2 THEN 'Low Value Recent'
  	ELSE 'Others'
END AS rfm_segment 

FROM scoring
)

SELECT *, -- groups segments together
	CASE
	WHEN rfm_segment IN ('Champions', 'Loyal Customers', 'Big Spenders') THEN 'High Value'
	WHEN rfm_segment IN ('Potential Loyalists', 'New Customers') THEN 'Growth'
	WHEN rfm_segment IN('At Risk') THEN 'High Risk'
	ELSE 'Low Engagement'
END AS segment_priority

FROM segments;

ALTER TABLE fact_customer_rfm_monthly
ADD COLUMN month_2_snapshot_date DATE,
ADD COLUMN month_2_rfm_segment TEXT,
ADD COLUMN month_2_segment_priority TEXT,

ADD COLUMN month_3_snapshot_date DATE,
ADD COLUMN month_3_rfm_segment TEXT,
ADD COLUMN month_3_segment_priority TEXT,

ADD COLUMN month_4_snapshot_date DATE,
ADD COLUMN month_4_rfm_segment TEXT,
ADD COLUMN month_4_segment_priority TEXT
;

WITH updates_1 AS (
  SELECT
    curr.customer_id,
    curr.snapshot_date,
    curr.snapshot_date + INTERVAL '1 month' AS next_snapshot_date,
    COALESCE(next.rfm_segment, 'Dropped Off') AS next_rfm_segment,
    COALESCE(next.segment_priority, 'Dropped Off') AS next_segment_priority
  FROM fact_customer_rfm_monthly curr
  LEFT JOIN fact_customer_rfm_monthly next
    ON curr.customer_id = next.customer_id
   AND curr.snapshot_date + INTERVAL '1 month' = next.snapshot_date

)

UPDATE fact_customer_rfm_monthly curr
SET 
    month_2_snapshot_date = u1.next_snapshot_date,
    month_2_rfm_segment = u1.next_rfm_segment,
    month_2_segment_priority = u1.next_segment_priority
FROM updates_1 u1
WHERE curr.customer_id = u1.customer_id
  AND curr.snapshot_date = u1.snapshot_date;


WITH updates_2 AS (
  SELECT
    curr.customer_id,
    curr.snapshot_date,
    curr.snapshot_date + INTERVAL '2 month' AS next_snapshot_date,
    COALESCE(next.rfm_segment, 'Dropped Off') AS next_rfm_segment,
    COALESCE(next.segment_priority, 'Dropped Off') AS next_segment_priority
  FROM fact_customer_rfm_monthly curr
  LEFT JOIN fact_customer_rfm_monthly next
    ON curr.customer_id = next.customer_id
   AND curr.snapshot_date + INTERVAL '2 month' = next.snapshot_date
 )
 
UPDATE fact_customer_rfm_monthly curr
SET 
    month_3_snapshot_date = u2.next_snapshot_date,
    month_3_rfm_segment = u2.next_rfm_segment,
    month_3_segment_priority = u2.next_segment_priority
FROM updates_2 u2
WHERE curr.customer_id = u2.customer_id
  AND curr.snapshot_date = u2.snapshot_date;

	
WITH updates_3 AS (
  SELECT
    curr.customer_id,
    curr.snapshot_date,
    curr.snapshot_date + INTERVAL '3 month' AS next_snapshot_date,
    COALESCE(next.rfm_segment, 'Dropped Off') AS next_rfm_segment,
    COALESCE(next.segment_priority, 'Dropped Off') AS next_segment_priority
  FROM fact_customer_rfm_monthly curr
  LEFT JOIN fact_customer_rfm_monthly next
    ON curr.customer_id = next.customer_id
   AND curr.snapshot_date + INTERVAL '3 month' = next.snapshot_date
)
  
UPDATE fact_customer_rfm_monthly curr
SET 
    month_4_snapshot_date = u3.next_snapshot_date,
    month_4_rfm_segment = u3.next_rfm_segment,
    month_4_segment_priority = u3.next_segment_priority
FROM updates_3 u3
WHERE curr.customer_id = u3.customer_id
  AND curr.snapshot_date = u3.snapshot_date;

Logic for monetary threshold:
	Query: 
	WITH cte AS (
  	SELECT 
    	d.month_label, 
    	c.customer_id, 
    	SUM(f.total_amount) AS cust_rev
  	FROM dim_customer c
  	JOIN fact_sales f ON c.customer_id = f.customer_id
  	JOIN dim_date d ON d.date = f.invoice_date
  	GROUP BY d.month_label, c.customer_id
	)

	SELECT 
  	PERCENTILE_CONT(0.8) WITHIN GROUP (ORDER BY cust_rev) AS p80_monthly_spend
	FROM cte;

...
```

## Dim Geography

Maps countries to regions and adds customer counts per country.

**Key Steps:**
- Counts customers by country.
- Flags countries with 20+ customers.

```sql

CREATE TABLE dim_geography AS 

WITH geo AS ( -- creates raw table with country id
SELECT DISTINCT country, 
		ROW_NUMBER() OVER(ORDER BY country) AS country_id,
		region, 
		continent
  FROM country_region_continent
),

customers_per_country AS ( 
SELECT g.country, COUNT(DISTINCT c.customer_id) AS customer_count
  FROM geo g
  JOIN dim_customer c ON g.country_id = c.country_id
GROUP BY g.country
)

SELECT g.*, -- outer query flags countries with 20+ customers. The join pulls this information into the raw geo table
	CASE WHEN cpc.customer_count >= 20 THEN 'Yes' ELSE 'No' END AS market_significant
  FROM geo g
  JOIN customers_per_country cpc ON g.country = cpc.country

...
```

## Dim Product

Maps product names to categories, labeling missing ones as 'Uncategorised'.

```sql
CREATE TABLE dim_product AS
SELECT DISTINCT fs.product_id, fs.product_name, 
    CASE WHEN pc.category IS NULL THEN 'Uncategorised' ELSE pc.category END pc.category
FROM fact_sales fs
LEFT JOIN product_categories_new pc ON fs.product_name = pc.product;
```

## Dim Date

Generates a full calendar table from Dec 2010 to Dec 2011 with useful time dimensions.

```sql
CREATE TABLE dim_date AS
WITH cte AS (
SELECT generate_series('2010-12-01'::date, '2011-12-31'::date, '1 day')::date AS Date
)
SELECT DISTINCT 
	Date,
	EXTRACT(DAY FROM Date) AS day,
	EXTRACT(MONTH FROM Date) AS month,
	EXTRACT(YEAR FROM Date) AS year,
	EXTRACT(DOW FROM Date) AS day_of_week,
	TO_CHAR(Date, 'Day') AS weekday_name,
	TO_CHAR(Date, 'Month') AS month_name,
	CASE WHEN EXTRACT(MONTH FROM Date) BETWEEN 1 AND 3 THEN 'Q1'
	WHEN EXTRACT(MONTH FROM Date) BETWEEN 4 AND 6 THEN 'Q2'
	WHEN EXTRACT(MONTH FROM Date) BETWEEN 7 AND 9 THEN 'Q3'
	WHEN EXTRACT(MONTH FROM Date) BETWEEN 10 AND 12 THEN 'Q4'
	END AS quarter
	
  FROM cte;


ALTER TABLE dim_date
ADD COLUMN quarter_label TEXT;
UPDATE dim_date
SET quarter_label = quarter ||' '|| year
```
