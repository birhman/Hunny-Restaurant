### 1. What is the total amount each customer spent at the restaurant?

```sql
SELECT s.customer_id, SUM(m.price) AS total_spent
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY s.customer_id;
```

### 2. How many days has each customer visited the restaurant?

```sql
SELECT customer_id, COUNT(DISTINCT order_date) AS days_visited
FROM sales
GROUP BY customer_id;
```

### 3. What was the first item from the menu purchased by each customer?

```sql
WITH ranked_orders AS (
    SELECT s.customer_id, m.product_name, s.order_date,
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS r
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
)
SELECT customer_id, product_name AS first_product
FROM ranked_orders
WHERE r = 1;
```

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT product_name, COUNT(*) AS total_purchases
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY total_purchases DESC
LIMIT 1;
```

### 5. Which item was the most popular for each customer?

```sql
WITH ranking AS (
    SELECT s.customer_id AS Customer, m.product_name AS Product, 
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) AS r
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
    GROUP BY s.customer_id, s.product_id, m.product_name
)
SELECT Customer, Product
FROM ranking WHERE r = 1; 
```

### 6. Which item was purchased first by the customer after they became a member?

```sql
WITH first_order AS (
    SELECT s.customer_id, s.product_id, mb.join_date, s.order_date,
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS num
    FROM sales s
    JOIN members mb ON s.customer_id = mb.customer_id AND mb.join_date < s.order_date
)
SELECT f.customer_id, m.product_name, f.join_date, order_date
FROM first_order f
JOIN menu m ON f.product_id = m.product_id
WHERE num = 1
ORDER BY f.customer_id;
```

### 7. Which item was purchased just before the customer became a member?

```sql
WITH first_order AS (
    SELECT s.customer_id, s.product_id, mb.join_date, s.order_date,
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS num
    FROM sales s
    JOIN members mb ON s.customer_id = mb.customer_id AND mb.join_date > s.order_date
)
SELECT f.customer_id, m.product_name, f.join_date, order_date
FROM first_order f
JOIN menu m ON f.product_id = m.product_id
WHERE num = 1
ORDER BY f.customer_id;
```

### 8. What is the total items and amount spent for each member before they became a member?

```sql
WITH first_order AS (
    SELECT s.customer_id, s.product_id, mb.join_date, s.order_date,
           COUNT(s.product_id) AS product_count
    FROM sales s
    JOIN members mb ON s.customer_id = mb.customer_id AND mb.join_date > s.order_date
    GROUP BY s.customer_id, s.product_id, mb.join_date, s.order_date
)
SELECT f.customer_id,
       SUM(f.product_count) AS total_items_purchased,
       SUM(f.product_count * m.price) AS total_amount_spent
FROM first_order f
JOIN menu m ON f.product_id = m.product_id
GROUP BY f.customer_id;
```

### 9. If each ₹1 spent equates to 1 points and biryani has a 2x points multiplier - how many points would each customer have?

```sql
WITH first_order AS (
    SELECT s.customer_id, s.product_id,
           COUNT(s.product_id) AS product_count
    FROM sales s
    JOIN members mb ON s.customer_id = mb.customer_id
    GROUP BY s.customer_id, s.product_id
)
SELECT f.customer_id,
       SUM(f.product_count * m.price) AS total_amount_spent,
       SUM(CASE WHEN f.product_id = '1' THEN 2 * (f.product_count * m.price)
                ELSE (f.product_count * m.price)
           END) AS total_points
FROM first_order f
JOIN menu m ON f.product_id = m.product_id
GROUP BY f.customer_id;
```

### 10. In the first week after a customer joins the program (including their join date), they earn 2x points on all items, not just biryani - how many points do customer A and B have at the end of January?

```sql
SELECT 
    s.customer_id,
    SUM(
        CASE 
            WHEN s.order_date BETWEEN mb.join_date AND DATE_ADD(mb.join_date, INTERVAL 6 DAY)
            THEN 2 * m.price
            ELSE m.price
        END
    ) AS total_points
FROM sales s
JOIN members mb ON s.customer_id = mb.customer_id
JOIN menu m ON s.product_id = m.product_id
WHERE s.order_date <= '2021-01-31'
GROUP BY s.customer_id
ORDER BY mb.customer_id;
```
