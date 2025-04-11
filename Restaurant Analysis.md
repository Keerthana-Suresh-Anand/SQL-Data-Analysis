# Restaurant Order Analysis

## [Dataset link](https://mavenanalytics.io/data-playground?order=date_added%2Cdesc&search=restaurant)
## Problem Statement
Help the restaurant owner understand how their business is doing, what the most popular dishes on the menu are, and how much revenue they are bringing in

## Understanding the tables
### Menu table
**1. How many items are on the menu?**
```sql
SELECT COUNT(item_name) AS no_of_items_on_menu FROM menu_items;
```
|no_of_items_on_menu|
|-----------|
|32|

- There are 32 items on the menu
---
**2. How many categories of items?**
```sql
SELECT COUNT(DISTINCT category) AS categories FROM menu_items;
```
|categories|
|-----------|
|4|

- There are 4 categories of items on the menu

---
**3. How many items are in each category?**
```sql
SELECT category, COUNT(*) AS no_of_items
FROM menu_items
GROUP BY category
ORDER BY no_of_items DESC;
```
|category|no_of_items|
|-----------|-------|
|Italian|9|
|Mexican|9|
|Asian|8|
|American|6|

- Italian and Mexican cuisines have the highest number of menu items, with 9 each, while American has the least with 6.

---

**4. Avg price of each category?**
```sql
SELECT category, CAST(AVG(price) AS DECIMAL(10,2)) AS avg_price
FROM menu_items
GROUP BY category
ORDER BY avg_price DESC;
```
|category|avg_price|
|-----------|-------|
|Italian|16.75|
|Asian|13.48|
|Mexican|11.80|
|American|10.07|

- Italian is the most expensive category

---
**5. Most and least expensive items and their category?**
```sql
SELECT category, item_name, price
FROM menu_items
WHERE price = (SELECT MAX(price) FROM menu_items);

SELECT category, item_name, price
FROM menu_items
WHERE price = (SELECT MIN(price) FROM menu_items);
```
|category|item_name|price|
|-----------|-------|---|
|Italian|Shrimp Scampi|19.95|

|category|item_name|price|
|-----------|-------|--|
|Asian|Edamame|5.00|
---
**6. Price range of each category?**
```sql
SELECT category, MIN(price)  AS min_price, MAX(price) AS max_price
FROM menu_items
GROUP BY category;
```
|category|min_price|max_price|
|-----------|-------|--|
|American|7.00|13.95|
|Asian|5.00|17.95|
|Italian|14.50|19.95|
|Mexican|7.00|14.95|
---
### Orders table
**1. Data range of the data**
```sql
SELECT MIN(order_date) AS min_date, MAX(order_date) AS max_date FROM order_details;
```
|min_date|max_date|
|-----------|---|
|2023-01-01|2023-03-31|
---

**2. Total orders**
```sql
SELECT COUNT(DISTINCT order_id) AS total_orders FROM order_details;
```
|total_orders|
|-----------|
|5370|
---

**3. Total items purchased**
```sql
SELECT COUNT(*) AS total_items FROM order_details;
```
|total_items|
|-----------|
|12234|
---

**4. Avg number of items in an order**
```sql
SELECT CAST(COUNT(order_id)*1.00/COUNT(DISTINCT order_id) AS DECIMAL(10,1)) AS avg_items_per_order
FROM order_details;
```
|avg_items_per_order|
|-----------|
|2.3|
---

**5. Avg orders and items ordered per month**
```sql
SELECT COUNT(DISTINCT order_id)/COUNT(DISTINCT MONTH(order_date)) AS avg_orders, 
       COUNT(order_id)/COUNT(DISTINCT MONTH(order_date)) AS avg_items
FROM order_details;
```
|avg_orders|avg_items|
|-----------|--|
|1790|4078|
---

**6. Total orders by month**
```sql
SELECT DATEPART(MONTH, order_date) AS [month], 
       COUNT(DISTINCT order_id) AS total_orders, 
       COUNT(order_id) AS total_items
FROM order_details
GROUP BY DATEPART(MONTH, order_date)
ORDER BY [month];
```
|month|total_orders|total_items|
|-----------|--|---|
|1|1845|4156|
|2|1685|3892|
|3|1840|4186|
---

**7. Most orders by day of the week**
```sql 
SELECT DATENAME(WEEKDAY, order_date) AS [day], 
       DATEPART(WEEKDAY, order_date) AS week_numb,
       COUNT(DISTINCT order_id) AS total_orders
FROM order_details
GROUP BY DATENAME(WEEKDAY, order_date),
         DATEPART(WEEKDAY, order_date)
ORDER BY DATEPART(WEEKDAY,order_date);
```
|day|week_numb|total_orders|
|-----------|--|---|
|Sunday|1|796|
|Monday|2|885|
|Tuesday|3|766|
|Wednesday|4|682|
|Thursday|5|743|
|Friday|6|787|
|Saturday|7|711|

- The restaurant receives most orders on Mondays and Sundays, and the least on Wednesdays
---

## Analyzing Customer Behaviour
**1. Avg price of an order**
```sql
SELECT CAST(SUM(m.price)/COUNT(DISTINCT o.order_id) AS DECIMAL(10,2)) AS avg_order_price
FROM order_details o
LEFT JOIN menu_items m
  ON o.item_id = m.menu_item_id;
```
|avg_order_price|
|-----------|
|29.65|

- Average price of an order is $29.65
---
**2. Top 10 most expensive orders**
```sql
SELECT TOP 10 o.order_id, SUM(m.price) AS total_amount
FROM order_details o
LEFT JOIN menu_items m
  ON o.item_id = m.menu_item_id
GROUP BY o.order_id
ORDER BY total_amount DESC;
```
| order_id | total_amount |
|----------|---------|
| 440      | 192.15  |
| 2075     | 191.05  |
| 1957     | 190.10  |
| 330      | 189.70  |
| 2675     | 185.10  |
| 4482     | 184.50  |
| 1274     | 183.55  |
| 2188     | 182.65  |
| 3473     | 182.55  |
| 3583     | 179.60  |
---
**3. Details of the most expensive orders**
```sql
SELECT m.category, COUNT(m.item_name) AS numb_items
FROM order_details o
LEFT JOIN menu_items m
	ON o.item_id = m.menu_item_id
WHERE o.order_id IN (440, 2075, 1957, 330, 2675, 4482, 1274, 2188, 3473, 3583)
GROUP BY m.category
ORDER BY numb_items DESC;
```
|category|numb_items|
|-----------|-------|
|Italian|45|
|Mexican|34|
|American|25|
|Asian|31|

- Italian is the most ordered category among the highest value orders
---

**4. Total revenue & revenue per month**
```sql
SELECT SUM(m.price) AS total_revenue
FROM order_details o
JOIN menu_items m
	ON o.item_id = m.menu_item_id;
```
```sql
SELECT MONTH(o.order_date) AS [month], SUM(m.price) AS revenue
FROM order_details o
JOIN menu_items m
	ON o.item_id = m.menu_item_id
GROUP BY MONTH(o.order_date)
ORDER BY [month];
```
|total_revenue|
|-----------|
|159217.90|

|month|revenue|
|-----------|-------|
|1|53816.95|
|2|50790.35|
|3|54610.60|

---
**5. Avg revenue per month**
```sql
SELECT CAST(SUM(m.price)/COUNT(DISTINCT MONTH(o.order_date)) AS DECIMAL(10,2)) AS avg_revenue
FROM order_details o
JOIN menu_items m
  ON o.item_id = m.menu_item_id;
```
|avg_revenue|
|-----------|
|53072.63|

---
**6. Least and most ordered items**
```sql
SELECT m.category,
       m.item_name,
       COUNT(o.order_details_id) AS no_of_times_ordered,
       SUM(m.price) AS total_price
FROM order_details o
LEFT JOIN menu_items m
  ON o.item_id = m.menu_item_id
GROUP BY m.category, m.item_name
ORDER BY no_of_times_ordered DESC;
```
|category|item_name|no_of_times_ordered|total_price|
|-----------|--|---|---|
|American|Hamburger|622|8054.90|
|Asian|Edamame|620|3100.00|
|Asian|Korean Beef Bowl|588|10554.60|
|--|--|--|--|
|Asian|Potstickets|205|1845.00|
|Mexican|Chicken Tacos|123|1469.85|

- Hamburger was the most ordered item, while Chicken Tacos were the least ordered.
- Surprisingly, Korean Beef Bowls, which ranked #3 in order volume, generated the highest revenue among all items. Meanwhile, Chicken Tacos also brought in the least revenue, consistent with the low order count.
---
**7. Least and most ordered categories**
```sql
SELECT m.category, COUNT(o.order_details_id) AS no_of_times_ordered, SUM(m.price) AS total_revenue
FROM order_details o
LEFT JOIN menu_items m
	ON o.item_id = m.menu_item_id
GROUP BY m.category
ORDER BY no_of_times_ordered DESC;
```
| category | no_of_times_ordered | total_revenue |
|----------|--------------|----------------|
| Asian    | 3470        | 46720.65      |
| Italian  | 2948        | 49462.70      |
| Mexican  | 2945        | 34796.80      |
| American | 2734        | 28237.75      |
| NULL     | 137          | NULL          |

- Even though Italian is second in order volume, it is #1 in revenue
---
