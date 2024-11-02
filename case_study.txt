#Case Study Questions

##What is the total amount each customer spent at the restaurant?
SELECT s.customer_id, SUM(m.price) AS total_spent
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY s.customer_id;


##How many days has each customer visited the restaurant?
SELECT customer_id, COUNT(DISTINCT order_date) AS days_visited
FROM sales
GROUP BY customer_id;


##What was the first item from the menu purchased by each customer?
WITH ranked_orders AS (
    SELECT s.customer_id, m.product_name, s.order_date,
           ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS r
    FROM sales s
    JOIN menu m ON s.product_id = m.product_id
)
SELECT customer_id, product_name AS first_product
FROM ranked_orders
WHERE r = 1;


##What is the most purchased item on the menu and how many times was it purchased by all customers?
SELECT product_name, COUNT(*) AS total_purchases
FROM sales s
JOIN menu m ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY total_purchases DESC
LIMIT 1;


##Which item was the most popular for each customer?
with ranking as(
	select s.customer_id as Customer, m.product_name as Product, row_number() over(partition by s.customer_id order by count(s.product_id) desc) as r
    from sales s
    join menu m on s.product_id = m.product_id
    group by s.customer_id, s.product_id, m.product_name
)
select Customer, Product
from ranking where r = 1; 


##Which item was purchased first by the customer after they became a member?
with first_order as(
	select s.customer_id, s.product_id, mb.join_date, s.order_date,
    row_number() over(partition by s.customer_id order by s.order_date) as num
    from sales s
    join members mb on s.customer_id = mb.customer_id and mb.join_date < s.order_date
    )
    
    select f.customer_id, m.product_name, f.join_date, order_date
    from first_order f
    join menu m on f.product_id = m.product_id
    where num = 1
    order by f.customer_id;


##Which item was purchased just before the customer became a member?
with first_order as(
	select s.customer_id, s.product_id, mb.join_date, s.order_date,
    row_number() over(partition by s.customer_id order by s.order_date desc) as num
    from sales s
    join members mb on s.customer_id = mb.customer_id and mb.join_date > s.order_date
    )
    
    select f.customer_id, m.product_name, f.join_date, order_date
    from first_order f
    join menu m on f.product_id = m.product_id
    where num = 1
    order by f.customer_id;


##What is the total items and amount spent for each member before they became a member?
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


##If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
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


##In the first week after a customer joins the program (including their join date), they earn 2x points on all items, not just sushi - 
how many points do customer A and B have at the end of January?
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
WHERE s.order_date <= '2021-01-31' AND s.customer_id IN ('A', 'B')    -- Focusing on customers A and B
GROUP BY s.customer_id
order by mb.customer_id;
