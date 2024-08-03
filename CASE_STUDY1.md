## SQL Queries and Results

### 1. What is the total amount each customer spent at the restaurant?

```sql
SELECT customer_id, CONCAT('Rs.', SUM(price)) AS total_sum 
FROM dannys_diner.sales 
INNER JOIN dannys_diner.menu ON sales.product_id = menu.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

### Result
customer_id | total_sum
------------|-----------
A           | Rs.76     
B           | Rs.74     
C           | Rs.36     


### 2. How many days has each customer visited the restaurant?

**SQL Query:**
```sql
SELECT customer_id, COUNT(DISTINCT order_date) AS visits 
FROM dannys_diner.sales 
GROUP BY customer_id;
```
### Result
customer_id | visits
------------|--------
A           | 4      
B           | 6      
C           | 2      


### 3. What was the first item from the menu purchased by each customer?

**SQL Query:**
```sql
WITH cte AS
  (SELECT customer_id,
          order_date,
          product_name,
          DENSE_RANK() OVER(PARTITION BY s.customer_id
                            ORDER BY s.order_date) AS rank_num
   FROM dannys_diner.sales AS s
   JOIN dannys_diner.menu AS m ON s.product_id = m.product_id)
SELECT customer_id,
       product_name
FROM cte
WHERE rank_num = 1
GROUP BY customer_id,
         product_name;
```
### Result
customer_id | product_name
------------|-------------
A           | curry       
A           | sushi       
B           | curry       
C           | ramen    


### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

**SQL Query:**
```sql
WITH q_4 AS 
  (SELECT * 
   FROM dannys_diner.sales 
   INNER JOIN dannys_diner.menu ON sales.product_id = menu.product_id)
SELECT product_name, COUNT(product_name) AS total_purchases 
FROM q_4
GROUP BY product_name
ORDER BY total_purchases DESC
LIMIT 1;
```
### Result
product_name | total_purchases
-------------|-----------------
ramen        | 8               


### 5. Which item was the most popular for each customer?

**SQL Query:**
```sql
SELECT customer_id, product_name
FROM ( 
    SELECT sales.customer_id,
           menu.product_name,
           COUNT(menu.product_name) AS product_count,
           RANK() OVER (PARTITION BY sales.customer_id ORDER BY COUNT(menu.product_name) DESC) AS rank
    FROM dannys_diner.sales 
    INNER JOIN dannys_diner.menu ON sales.product_id = menu.product_id
    GROUP BY sales.customer_id, menu.product_name
) ranked
WHERE rank = 1;
```
### Result
customer_id | product_name
------------|-------------
A           | ramen       
B           | ramen       
B           | curry       
B           | sushi       
C           | ramen       


### 6. Which item was purchased first by the customer after they became a member?

**SQL Query:**
```sql
SELECT cte.customer_id, cte.product_name, cte.order_date
FROM (
    SELECT s.customer_id, m.product_name, s.order_date,
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS row_num
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id = m.product_id
    LEFT JOIN dannys_diner.members AS mb ON s.customer_id = mb.customer_id
    WHERE s.order_date > mb.join_date
) cte
WHERE cte.row_num = 1
ORDER BY cte.order_date;
```
### Result
customer_id | product_name | order_date
------------|--------------|-----------------------------
A           | ramen        | 2021-01-10T00:00:00.000Z
B           | sushi        | 2021-01-11T00:00:00.000Z


### 7. Which item was purchased just before the customer became a member?

**SQL Query:**
```sql
SELECT customer_id, product_name
FROM (
    SELECT s.customer_id, m.product_name, s.order_date,
           RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS row_num
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id = m.product_id
    LEFT JOIN dannys_diner.members AS mb ON s.customer_id = mb.customer_id
    WHERE s.order_date < mb.join_date
) cte
WHERE row_num = 1
ORDER BY customer_id;
```
### Result
customer_id | product_name
------------|-------------
A           | sushi       
A           | curry       
B           | sushi       


### 8. What is the total items and amount spent for each member before they became a member?

**SQL Query:**
```sql
WITH cte AS (
    SELECT s.*, m.product_name, m.price, 
           CASE 
               WHEN mb.join_date <= s.order_date THEN 'Y'
               WHEN mb.join_date > s.order_date THEN 'N'
               ELSE 'N' 
           END AS join_status
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id = m.product_id
    LEFT JOIN dannys_diner.members AS mb ON s.customer_id = mb.customer_id
)
SELECT cte.customer_id, COUNT(cte.product_id) AS product_count, SUM(cte.price) AS total_price
FROM cte
WHERE cte.join_status = 'N'
GROUP BY cte.customer_id
ORDER BY cte.customer_id;
```
### Result
customer_id | product_count | total_price
------------|---------------|------------
A           | 2             | 25         
B           | 3             | 40         
C           | 3             | 36         


### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

**SQL Query:**
```sql
WITH CTE AS (
    SELECT s.*, 
           CASE 
               WHEN m.product_name = 'sushi' THEN (m.price * 10) * 2 
               WHEN m.product_name = 'curry' THEN m.price * 10
               WHEN m.product_name = 'ramen' THEN m.price * 10 
           END AS points
    FROM dannys_diner.sales AS s
    LEFT JOIN dannys_diner.menu AS m ON s.product_id = m.product_id
)
SELECT customer_id, SUM(points) AS sum_of_points
FROM CTE
GROUP BY customer_id;
```
### Result
customer_id | sum_of_points
------------|--------------
A           | 860          
B           | 940          
C           | 360          


### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

**SQL Query:**
```sql
WITH cte AS (
    SELECT a.customer_id,
           CASE 
               WHEN c.product_name = 'sushi' THEN 2 * c.price
               WHEN a.order_date BETWEEN b.join_date AND b.join_date + INTERVAL '6 DAY' THEN 2 * c.price
               ELSE c.price 
           END AS newprice
    FROM dannys_diner.sales a
    JOIN dannys_diner.menu c ON a.product_id = c.product_id
    JOIN dannys_diner.members b ON a.customer_id = b.customer_id
    WHERE a.order_date <= '2021-01-31'
)
SELECT customer_id, SUM(newprice) * 10 AS total_points
FROM cte
GROUP BY customer_id
HAVING customer_id IN ('A', 'B');
```
### Result
customer_id | total_points
------------|-------------
A           | 1370        
B           | 820         







