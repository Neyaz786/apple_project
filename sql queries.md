## Queries and solution


### Apple Sales Project - 1M rows sales datasets

```sql
USE apple_db;

SELECT * FROM category;
SELECT * FROM products;
SELECT * FROM stores;
SELECT * FROM sales;
SELECT * FROM warranty;

```
##### EDA
```sql
SELECT DISTINCT repair_status FROM warranty;
SELECT COUNT(*) FROM sales;
```

###### Improving Query Performance 

-- et - 64.ms
-- pt - 0.15ms
-- et after index 5-10 ms

```sql
EXPLAIN
SELECT * FROM sales
WHERE product_id ='P-44';

CREATE INDEX sales_product_id ON sales(product_id);
CREATE INDEX sales_store_id ON sales(store_id);
CREATE INDEX sales_sale_date ON sales(sale_date);

```


-- et - 58.ms
-- pt - 0.069
-- et after index 2 ms
```sql

EXPLAIN
SELECT * FROM sales
WHERE store_id = 'ST-31';
```


#### 1. Find the number of stores in each country.
```sql
SELECT 
	country,
	COUNT(store_id) as total_stores
FROM stores
GROUP BY country
ORDER BY total_stores DESC;

```

#### Q.2 Calculate the total number of units sold by each store.
```sql
SELECT 
	s.store_id,
	st.store_name,
	SUM(s.quantity) as total_unit_sold
FROM sales as s
JOIN stores as st ON st.store_id = s.store_id
GROUP BY s.store_id, st.store_name
ORDER BY total_unit_sold DESC;
```
#### Q.3 Identify how many sales occurred in December 2023.
```sql
SELECT 
	COUNT(sale_id) as total_sale 
FROM sales
WHERE DATE_FORMAT(sale_date, '%m-%Y') = '12-2023';
```

#### Q.4 Determine how many stores have never had a warranty claim failed.
```sql
SELECT COUNT(*) FROM stores
WHERE store_id NOT IN (
	SELECT DISTINCT s.store_id
	FROM sales as s
	RIGHT JOIN warranty as w ON s.sale_id = w.sale_id
);

```

#### Q.5 Calculate the percentage of warranty claims marked as "Warranty Void".
```sql
SELECT 
	ROUND(
		(COUNT(claim_id)/(SELECT COUNT(*) FROM warranty)) * 100, 2
	) as warranty_void_percentage
FROM warranty
WHERE repair_status = 'Warranty Void';

```
#### Q.6 Identify which store had the highest total units sold in the last year.
```sql
SELECT 
	s.store_id,
	st.store_name,
	SUM(s.quantity) as total_qty
FROM sales as s
JOIN stores as st ON s.store_id = st.store_id
WHERE sale_date >= CURDATE() - INTERVAL 1 YEAR
GROUP BY s.store_id, st.store_name
ORDER BY total_qty DESC
LIMIT 1;
```
#### Q.7 Count the number of unique products sold in the last year.
```sql
SELECT 
	COUNT(DISTINCT product_id) as unique_products
FROM sales
WHERE sale_date >= CURDATE() - INTERVAL 1 YEAR;
```
#### Q.8 Find the average price of products in each category.
```sql
SELECT 
	p.category_id,
	c.category_name,
	AVG(p.price) as avg_price
FROM products as p
JOIN category as c ON p.category_id = c.category_id
GROUP BY p.category_id, c.category_name
ORDER BY avg_price DESC;
```
#### Q.9 How many warranty claims were filed in 2020?
```sql
SELECT 
	COUNT(*) as warranty_claim
FROM warranty
WHERE YEAR(claim_date) = 2020;
```
#### Q.10 For each store, identify the best-selling day based on highest quantity sold.

```sql
SELECT * FROM (
  SELECT 
    store_id,
    DAYNAME(sale_date) AS day_name,
    SUM(quantity) AS total_unit_sold,
    RANK() OVER (PARTITION BY store_id ORDER BY SUM(quantity) DESC) AS `rank`
  FROM sales
  GROUP BY store_id, day_name
) AS t1
WHERE `rank` = 1;
```

#### Q.11 Identify the least selling product in each country for each year based on total units sold.
```sql

SELECT
  st.country,
  p.product_name,
  SUM(s.quantity) AS total_qty_sold
FROM sales s
JOIN stores st ON s.store_id = st.store_id
JOIN products p ON s.product_id = p.product_id
GROUP BY st.country, p.product_name
HAVING SUM(s.quantity) = (
  SELECT MIN(total_qty)
  FROM (
    SELECT SUM(quantity) AS total_qty
    FROM sales s2
    JOIN stores st2 ON s2.store_id = st2.store_id
    WHERE st2.country = st.country
    GROUP BY s2.product_id
  ) AS country_totals
)
ORDER BY st.country, total_qty_sold;

```

#### Another solution

```sql
SELECT country, year, product_id, total_unit_sold
FROM (
  SELECT 
    st.country,
    YEAR(s.sale_date) AS year,
    s.product_id,
    SUM(s.quantity) AS total_unit_sold,
    RANK() OVER (
      PARTITION BY st.country, YEAR(s.sale_date) 
      ORDER BY SUM(s.quantity)
    ) AS rnk
  FROM sales s
  JOIN stores st ON s.store_id = st.store_id
  GROUP BY st.country, year, s.product_id
) t
WHERE rnk = 1;
```



#### Q.12 Calculate how many warranty claims were filed within 180 days of a product sale.
```sql
SELECT 
	COUNT(*) as claim_within_180
FROM warranty as w
LEFT JOIN sales as s ON s.sale_id = w.sale_id
WHERE DATEDIFF(w.claim_date, s.sale_date) <= 180;
```
#### Q.13 Determine how many warranty claims were filed for products launched in the last two years.

```sql
SELECT 
	p.product_name,
	COUNT(w.claim_id) as no_claim,
	COUNT(s.sale_id) as total_sales
FROM warranty as w
RIGHT JOIN sales as s ON s.sale_id = w.sale_id
JOIN products as p ON p.product_id = s.product_id
WHERE p.launch_date >= CURDATE() - INTERVAL 2 YEAR
GROUP BY p.product_name
HAVING no_claim > 0;
```
#### Q.14 List the months in the last three years where sales exceeded 5,000 units in the USA.
```sql

SELECT 
	DATE_FORMAT(sale_date, '%m-%Y') as month,
	SUM(s.quantity) as total_unit_sold
FROM sales as s
JOIN stores as st ON s.store_id = st.store_id
WHERE st.country = 'USA' AND s.sale_date >= CURDATE() - INTERVAL 3 YEAR
GROUP BY month
HAVING total_unit_sold > 5000;

```
#### Q.15 Identify the product category with the most warranty claims filed in the last two years.
```sql

SELECT 
	c.category_name,
	COUNT(w.claim_id) as total_claims
FROM warranty as w
LEFT JOIN sales as s ON w.sale_id = s.sale_id
JOIN products as p ON p.product_id = s.product_id
JOIN category as c ON c.category_id = p.category_id         -- loST CONNECTION
WHERE w.claim_date >= CURDATE() - INTERVAL 2 YEAR
GROUP BY c.category_name;

```



#### Q.16 Determine the percentage chance of receiving warranty claims after each purchase for each country!
```sql

SELECT 
	country,
	total_unit_sold,
	total_claim,
	ROUND(COALESCE(total_claim / total_unit_sold * 100, 0), 2) as risk
FROM (
	SELECT 
		st.country,
		SUM(s.quantity) as total_unit_sold,
		COUNT(w.claim_id) as total_claim
	FROM sales as s
	JOIN stores as st ON s.store_id = st.store_id
	LEFT JOIN warranty as w ON w.sale_id = s.sale_id
	GROUP BY st.country
) t1
ORDER BY risk DESC;
```
#### Q.17 Analyze the year-by-year growth ratio for each store.

```sql
WITH yearly_sales AS (
	SELECT 
		s.store_id,
		st.store_name,
		YEAR(sale_date) as year,
		SUM(s.quantity * p.price) as total_sale
	FROM sales as s
	JOIN products as p ON s.product_id = p.product_id
	JOIN stores as st ON st.store_id = s.store_id
	GROUP BY s.store_id, st.store_name, year
),
growth_ratio AS (
	SELECT 
		store_name,
		year,
		LAG(total_sale) OVER(PARTITION BY store_name ORDER BY year) as last_year_sale,
		total_sale as current_year_sale
	FROM yearly_sales
)
SELECT 
	store_name,
	year,
	last_year_sale,
	current_year_sale,
	ROUND((current_year_sale - last_year_sale) / last_year_sale * 100, 3) as growth_ratio
FROM growth_ratio
WHERE last_year_sale IS NOT NULL AND year <> YEAR(CURDATE());
```
#### Q.18 Calculate the correlation between product price and warranty claims for products sold in the last five years, segmented by price range.
```sql

SELECT 
	CASE
		WHEN p.price < 500 THEN 'Less Expenses Product'
		WHEN p.price BETWEEN 500 AND 1000 THEN 'Mid Range Product'
		ELSE 'Expensive Product'
	END as price_segment,
	COUNT(w.claim_id) as total_Claim
FROM warranty as w
LEFT JOIN sales as s ON w.sale_id = s.sale_id
JOIN products as p ON p.product_id = s.product_id
WHERE w.claim_date >= CURDATE() - INTERVAL 5 YEAR
GROUP BY price_segment;

```
#### Q.19 Identify the store with the highest percentage of "Paid Repaired" claims relative to total claims filed
```sql

WITH paid_repair AS (
	SELECT 
		s.store_id,
		COUNT(w.claim_id) as paid_repaired
	FROM sales as s
	RIGHT JOIN warranty as w ON w.sale_id = s.sale_id
	WHERE w.repair_status = 'Paid Repaired'
	GROUP BY s.store_id
),
total_repaired AS (
	SELECT 
		s.store_id,
		COUNT(w.claim_id) as total_repaired
	FROM sales as s
	RIGHT JOIN warranty as w ON w.sale_id = s.sale_id
	GROUP BY s.store_id
)
SELECT 
	tr.store_id,
	st.store_name,
	pr.paid_repaired,
	tr.total_repaired,
	ROUND(pr.paid_repaired / tr.total_repaired * 100, 2) as percentage_paid_repaired
FROM paid_repair as pr
JOIN total_repaired tr ON pr.store_id = tr.store_id
JOIN stores as st ON tr.store_id = st.store_id;

```
-- Q.20 Monthly running total of sales for each store over the past four years.

```sql
WITH monthly_sales AS (
	SELECT 
		store_id,
		YEAR(sale_date) as year,
		MONTH(sale_date) as month,
		SUM(p.price * s.quantity) as total_revenue
	FROM sales as s
	JOIN products as p ON s.product_id = p.product_id
	WHERE sale_date >= CURDATE() - INTERVAL 4 YEAR
	GROUP BY store_id, year, month
)
SELECT 
	store_id,
	month,
	year,
	total_revenue,
	SUM(total_revenue) OVER(PARTITION BY store_id ORDER BY year, month) as running_total
FROM monthly_sales;

```
