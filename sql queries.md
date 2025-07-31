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

#### Q.1 Find the number of stores in each country.
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
WHERE sale_date BETWEEN '2023-12-01' AND '2023-12-31';
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
		(COUNT(claim_id) * 100 /(SELECT COUNT(*) FROM warranty)), 2
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
JOIN category as c ON c.category_id = p.category_id        
WHERE w.claim_date >= CURDATE() - INTERVAL 2 YEAR
GROUP BY c.category_name;

```



