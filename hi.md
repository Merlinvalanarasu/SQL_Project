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
