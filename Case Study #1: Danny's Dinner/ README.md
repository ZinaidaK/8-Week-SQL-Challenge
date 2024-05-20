
# 🍜 Case Study #1: Danny's Diner

![image](https://github.com/ZinaidaK/8-Week-SQL-Challenge/assets/100050035/c0a88634-d1f3-4354-bf2c-103adf853a17)


## 📚 Table of Contents
+ Business Task
+ Entity Relationship Diagram
+ Case Study Questions And Solution

## Business Task

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

## Case Study Questions And Solution

### 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT 
  s.customer_id, 
  SUM(m.price) AS total_amount
FROM dannys_diner.sales as s
INNER JOIN dannys_diner.menu as m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id ASC;
```

### Answer
| customer_id | total_amount |
| ----------- | ------------ |
| Customer A  |   76         |
| Customer B  |   74         |
| Customer C  |   36         |


## 2. How many days has each customer visited the restaurant?

```sql
SELECT 
  customer_id, 
  COUNT(DISTINCT order_date) AS total_days
FROM dannys_diner.sales
GROUP BY customer_id;
```

### Answer
| customer_id | total_days |
| ----------- | ---------- |
| Customer A  |   4        |
| Customer B  |   6        |
| Customer C  |   2        |


## 3.What was the first item(s) from the menu purchased by each customer?

```sql
WITH CTE AS 
	(
  	SELECT s.customer_id,  m.product_name, s.order_date,
  	DENSE_RANK() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rnk
  	FROM dannys_diner.sales AS s
  	JOIN dannys_diner.menu AS m
  	ON s.product_id = m.product_id
  	)
  
  SELECT
  	customer_id, product_name
    FROM CTE
    WHERE rnk = 1
    GROUP BY customer_id, product_name;
```

## Answer
| customer_id | product_name |
| ----------- | -------------|
| Customer A  |   curry      |
| Customer A  |   sushi      |
| Customer B  |   curry      |
| Customer C  |   ramen      |


## 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT
	m.product_name, 
	COUNT(s.product_id) AS most_purchased_product
FROM dannys_diner.sales as s
JOIN dannys_diner.menu as m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY most_purchased_product DESC
LIMIT 1;
```

### Answer
| product_name  | most_purchased_product |
| ------------- | -----------------------|
| ramen         |   8                    |


## 5. Which item was the most popular for each customer?

```sql
WITH CTE AS (
  
SELECT s.customer_id,
  m.product_name, 
  COUNT(m.product_id) AS order_count,
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(m.product_id)DESC) AS order_rank 
FROM dannys_diner.sales s 
JOIN dannys_diner.menu m 
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_name
)
SELECT customer_id, 
product_name, 
order_count
FROM CTE
WHERE order_rank = 1;
```

## Answer
| customer_id | product_name | order_count |
| ----------- | -------------| ------------|
| Customer A  |   ramen      |    3        |
| Customer B  |   sushi      |    2        |
| Customer B  |   curry      |    2        |
| Customer B  |   ramen      |    2        |
| Customer C  |   ramen      |    3        |


## 6. Which item was purchased just before the customer became a member?

```sql
WITH member_sales_cte AS (
  SELECT
    sales.customer_id,
    sales.order_date,
    menu.product_name,
    RANK() OVER (
      PARTITION BY sales.customer_id
      ORDER BY sales.order_date
    ) AS order_rank
  FROM dannys_diner.sales
  INNER JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
  INNER JOIN dannys_diner.members
    ON sales.customer_id = members.customer_id
  WHERE
    sales.order_date >= members.join_date
)
SELECT
  customer_id,
  order_date,
  product_name
FROM member_sales_cte
WHERE order_rank = 1
ORDER BY customer_id;
```

## Answer
| customer_id |  order_date  | product_name|
| ----------- | -------------| ------------|
| Customer A  | 2021-01-07   |    curry    |
| Customer B  |   sushi      |    sushi    |


## 7.Which menu item(s) was purchased just before the customer became a member and when?

```sql
WITH pre_membership_sales AS (
  SELECT
    s.customer_id,
    s.order_date,
    m.product_name,
    s.product_id,
    mem.join_date,
    RANK() OVER (
      PARTITION BY s.customer_id
      ORDER BY s.order_date DESC
    ) AS rank_before_join
  FROM
    dannys_diner.sales s
    JOIN dannys_diner.menu m ON s.product_id = m.product_id
    JOIN dannys_diner.members mem ON s.customer_id = mem.customer_id
  WHERE
    s.order_date < mem.join_date
)
SELECT
  customer_id,
  order_date,
  product_name
FROM
  pre_membership_sales
WHERE
  rank_before_join = 1
ORDER BY
  customer_id;
```

## Answer

| customer_id |  order_date  | product_name|
| ----------- | -------------| ------------|
| Customer A  | 2021-01-01   |    sushi    |
| Customer A  | 2021-01-01   |    curry    |
| Customer B  | 2021-01-04   |    sushi    |


## 8. What is the number of unique menu items and total amount spent for each member before they became a member?

```sql
SELECT 
  sales.customer_id, 
  COUNT(DISTINCT sales.product_id) AS unique_menu_items, 
  SUM(menu.price) AS total_amount_spent
FROM dannys_diner.sales
INNER JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
  AND sales.order_date < members.join_date
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

##  Answer
| customer_id | unique_menu_items | total_amount_spent |
| ----------- | ------------------| -------------------|
| Customer A  |      2            |       25           |
| Customer A  |      2            |       40           |


## 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

```sql
WITH sales_points AS (
  SELECT
    s.customer_id,
    s.order_date,
    s.product_id,
    m.price,
    m.product_name,
    CASE 
      WHEN m.product_name = 'sushi' THEN (m.price * 10 * 2)
      ELSE (m.price * 10)
    END AS points
  FROM
    dannys_diner.sales s
    JOIN dannys_diner.menu m ON s.product_id = m.product_id
)
SELECT
  sp.customer_id,
  SUM(sp.points) AS total_points
FROM
  sales_points sp
GROUP BY
  sp.customer_id
ORDER BY
  sp.customer_id;
```

## Answer

| customer_id |  total_points | 
| ----------- | --------------|
| Customer A  |  860          | 
| Customer A  |  940          |
| Customer C  |  360          |

## 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

```sql
WITH dates_cte AS (
  SELECT 
    customer_id, 
    join_date, 
    join_date + interval '6 days' AS valid_date,
    '2021-01-31'::DATE AS last_date
  FROM dannys_diner.members
)

SELECT 
  s.customer_id, 
  SUM(
    CASE
      WHEN s.order_date BETWEEN d.join_date AND d.valid_date THEN 2 * 10 * m.price
      WHEN m.product_name = 'sushi' THEN 2 * 10 * m.price
      ELSE 10 * m.price 
    END
  ) AS points
FROM dannys_diner.sales s
INNER JOIN dates_cte d
  ON s.customer_id = d.customer_id
INNER JOIN dannys_diner.menu m
  ON s.product_id = m.product_id
WHERE s.order_date <= '2021-01-31'
GROUP BY s.customer_id
HAVING s.customer_id IN ('A', 'B')
ORDER BY s.customer_id;
```

## Answer
| customer_id |  points | 
| ----------- | --------------|
| Customer A  |  1370         | 
| Customer B  |  820          |


## Bonus Question 11
Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL Recreate the following table output using the available data.

```sql
SELECT
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE
    WHEN s.order_date >= mem.join_date THEN 'Y'
    ELSE 'N'
  END AS member
FROM
  dannys_diner.sales s
JOIN dannys_diner.menu m ON s.product_id = m.product_id
LEFT JOIN dannys_diner.members mem ON s.customer_id = mem.customer_id
ORDER BY
  s.customer_id, s.order_date;
```

## Answer
![Bonus Question](https://github.com/ZinaidaK/8-Week-SQL-Challenge/assets/100050035/45086386-4b3d-4ccc-8717-150364c98470)

## Bonus Question 12
Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

```sql
WITH member_status_cte AS (
  SELECT
    s.customer_id,
    s.order_date,
    m.product_name,
    m.price,
    CASE
      WHEN s.order_date >= mem.join_date THEN 'Y'
      ELSE 'N'
    END AS is_member
  FROM
    dannys_diner.sales s
    LEFT JOIN dannys_diner.members mem ON s.customer_id = mem.customer_id
    JOIN dannys_diner.menu m ON s.product_id = m.product_id
)

SELECT
  ms.customer_id,
  ms.order_date,
  ms.product_name,
  ms.price,
  ms.is_member,
  CASE
    WHEN ms.is_member = 'Y' THEN RANK() OVER (PARTITION BY ms.customer_id ORDER BY ms.order_date)
    ELSE NULL
  END AS product_rank
FROM
  member_status_cte ms
ORDER BY
  ms.customer_id,
  ms.order_date;
```

## Answer
![bonusQuestion 12](https://github.com/ZinaidaK/8-Week-SQL-Challenge/assets/100050035/be0a7f6d-4b60-46d8-8cf0-b88f5d59b9cb)

### Explanation:

The member_status_cte determines whether each sale was made before or after the customer joined the loyalty program by checking if the order_date is on or after the join_date. It adds an is_member column with values 'Y' or 'N'.

Main Query:
I used the RANK() window function to rank purchases for each customer, but only for those where is_member = 'Y'. For non-member purchases, the rank is set to NULL.
The CASE statement checks the is_member status and applies the rank conditionally.
Then I ordered the results by customer_id and order_date for clarity.
This query provides the ranking of customer products, with NULL values for the rankings of non-member purchases.


