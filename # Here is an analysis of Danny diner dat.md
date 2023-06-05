# Here is an analysis of Danny diner data

## Question 1: What is the total amount each customer spent at the restaurant?

```` sql
WITH cte AS (
  SELECT s.customer_id, m.product_id, m.price
  FROM sales s
  JOIN menu m ON s.product_id = m.product_id
)
SELECT customer_id, SUM(price) AS total_spent
FROM cte
GROUP BY customer_id
ORDER BY total_spent DESC;
````

** Result **
| customer_id | total_spent |
|-------------|-------------|
| A           | 76          |
| B           | 74          |
| C           | 36          |

## Question 2: How many days has each customer visited the restaurant?

````sql
select customer_id, count(distinct order_date) as num_of_days from sales
group by customer_id
order by num_of_days desc
````

** Result **
| customer_id | total_spent | num_of_days |
|-------------|-------------|-------------|
| A           | 76          | 4           |
| B           | 74          | 6           |
| C           | 36          | 2           |

## Question 3: What was the first item from the menu purchased by each customer?

````sql

with cte as
(select sl.customer_id, sl.order_date, mn.product_name, ROW_NUMBER() over (partition by customer_id order by order_date) as rn
from sales sl join menu mn on sl.product_id = mn.product_id)
select customer_id, product_name from cte
where rn=1
````

** Result **
| customer_id | product_name |
|-------------|--------------|
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

## Question 4: What is the most purchased item on the menu and how many times was it purchased by all customers?

````sql

with cte as (
select sl.customer_id, sl.order_date, sl.product_id, mn.product_name from sales sl join menu mn on sl.product_id = mn.product_id
)
select top 1 product_name, COUNT(*) as num_purchased_times from cte
group by product_name
order by num_purchased_times desc
````
** Result ** 
| product_name | num_purchased_times|
|--------------|--------------------|
| ramen        | 8                  |

## Question 5: Which item was the most popular for each customer?

````sql

WITH cte AS (
  SELECT sl.customer_id, sl.order_date, sl.product_id, mn.product_name
  FROM sales sl
  JOIN menu mn ON sl.product_id = mn.product_id
)
SELECT customer_id, product_name, n_times
FROM (
  SELECT customer_id, product_name, COUNT(product_name) AS n_times,
         DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY COUNT(product_name) DESC) AS rn
  FROM cte
  GROUP BY customer_id, product_name
) AS subquery
WHERE rn = 1
ORDER BY n_times DESC;
````

** Result **
| customer_id | product_name | n_times |
|-------------|--------------|---------|
| A           | ramen        | 3       |
| C           | ramen        | 3       |
| B           | sushi        | 2       |
| B           | curry        | 2       |
| B           | ramen        | 2       |


## Question 6: Which item was purchased first by the customer after they became a member?

````sql

with cte as (
	select sl.customer_id, sl.order_date, mb.join_date, mn.product_name, 
			datediff(day,mb.join_date, sl.order_date) as num_days,	
			DENSE_RANK() over(partition by sl.customer_id order by datediff(day,mb.join_date, sl.order_date) asc) as rn
	from sales sl join members mb on sl.customer_id = mb.customer_id join menu mn on mn.product_id = sl.product_id
	where  datediff(day,mb.join_date, sl.order_date) >= 0
)
select customer_id, product_name from cte
where rn=1
````
** Result **
| customer_id | product_name |
|-------------|--------------|
| A           | curry        |
| B           | sushi        |

## Question 7: Which item was purchased just before the customer became a member?

````sql

with cte as (
	select sl.customer_id, sl.order_date, mb.join_date, mn.product_name, 
			datediff(day,mb.join_date, sl.order_date) as num_days,	
			DENSE_RANK() over(partition by sl.customer_id order by datediff(day,mb.join_date, sl.order_date) desc) as rn
	from sales sl join members mb on sl.customer_id = mb.customer_id join menu mn on mn.product_id = sl.product_id
	where  datediff(day,mb.join_date, sl.order_date) <= 0
)
select customer_id, product_name from cte
where rn=1
````
** Result **
| customer_id | product_name |
|-------------|--------------|
| A           | curry        |
| B           | sushi        |

## Question 8: What is the total items and amount spent for each member before they become a member?

````sql

with cte as (
SELECT sl.customer_id,mn.product_name,mn.price, coalesce(datediff(day,mb.join_date,sl.order_date),-3) as not_yet
  FROM sales AS sl
  FULL JOIN members AS mb ON sl.customer_id = mb.customer_id
  FULL JOIN menu AS mn ON sl.product_id = mn.product_id
  where coalesce(datediff(day,mb.join_date,sl.order_date),-3) < 0
  )
  select customer_id, count(distinct product_name) as total_items, SUM(price) as total_price from cte
  group by customer_id
````

** Result**
| customer_id | total_items | total_price |
|-------------|-------------|-------------|
| A           | 2           | 25          |
| B           | 2           | 40          |
| C           | 1           | 36          |

## Question 9: If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?````sql

````sql

  SELECT sl.customer_id, SUM(price * CASE 
                                    WHEN mn.product_name = 'sushi' THEN 20
                                    ELSE 10
                                 END) AS accumulated_point
FROM sales sl
JOIN menu mn ON sl.product_id = mn.product_id
GROUP BY sl.customer_id;
````
** Result**
| customer_id | accumulated_point |
|-------------|------------------|
| A           | 860              |
| B           | 940              |
| C           | 360              |

## Question 10: In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi how many points do customer A and B have at the end of January?

````sql

WITH cte AS (
    SELECT sl.customer_id, mn.product_name, mn.price, DATEDIFF(DAY, mb.join_date, sl.order_date) AS datediff
    FROM sales AS sl
    JOIN members AS mb ON sl.customer_id = mb.customer_id
    JOIN menu AS mn ON sl.product_id = mn.product_id
)
SELECT customer_id, SUM(price *
    CASE
        WHEN datediff >= 0 AND datediff <= 7 THEN
            CASE
                WHEN product_name = 'sushi' THEN 20 * 2
                ELSE 10 * 2
            END
        ELSE
            CASE
                WHEN product_name = 'sushi' THEN 20
                ELSE 10
            END
    END
) AS accumulated_points
FROM cte
GROUP BY customer_id;
````
** Result**
| customer_id | accumulated_points |
|-------------|-------------------|
| A           | 1370              |
| B           | 1260              |
