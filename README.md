# SQL Practice  

## Understanding the Database Structure  

To get a good handle on the database, it’s important to know how the tables are structured and connected. The dataset has four tables: `orders`, `order_details`, `pizzas`, and `pizza_types`. Each of these tables is key to understanding pizza sales and customer habits.

- **Orders**: This table logs information about each order placed.  
- **Order_Details**: Here, you’ll find specifics about each order, including which pizzas were ordered and in what quantities.  
- **Pizzas**: This table lists details about the various pizzas available.  
- **Pizza_Types**: This table classifies the pizzas and provides descriptions for each type.  

### Connections Between Tables  
The connections between these tables are crucial for accurate analysis. Here’s how they are linked:

- The `orders` table connects to the `order_details` table through the `order_id`.  
- The `order_details` table connects to the `pizzas` table through the `pizza_id`.  
- The `pizzas` table connects to the `pizza_types` table through the `pizza_type_id`.  
These links allows tables to be connected.

## Importing the library and Data

```r
# Importing the library
suppressWarnings({library(sqldf)})
# Loading required packages: gsubfn, proto, RSQLite

# Importing the data
orders <- read.csv("D:/SQL Practice/orders.csv", header = T)
order_details <- read.csv("D:/SQL Practice/order_details.csv", header = T)
pizzas <- read.csv("D:/SQL Practice/pizzas.csv", header = T)
pizza_types <- read.csv("D:/SQL Practice/pizza_types.csv", header = T)
```

### Properties of the data

```r
str(orders)
summary(orders)

str(order_details)
summary(order_details)

str(pizzas)
summary(pizzas)

str(pizza_types)
summary(pizza_types)
```

### Check Nulls Before Joins

```r
sum(is.na(orders))
sum(is.na(order_details))
sum(is.na(pizzas))
sum(is.na(pizza_types))
```

As there are no any null values so when we will need join we will just apply the inner joins.

---

## Now running the queries

### Q#1: How many orders are placed in total?

```r
q1 <- sqldf("SELECT COUNT(order_id) AS total_orders FROM orders;")
q1
# total_orders = 21350
```

### Q#2: Calculate the total revenue generated from pizza sales.

```r
q2 <- sqldf("SELECT ROUND(SUM(order_details.quantity * pizzas.price), 2) AS total_revenue
             FROM order_details
             JOIN pizzas ON pizzas.pizza_id = order_details.pizza_id;")
q2
# total_revenue = 817860.1
```

### Q#3: Identify the highest-priced pizza.

```r
q3 <- sqldf("SELECT pizza_types.name, pizzas.price
             FROM pizza_types
             JOIN pizzas ON pizza_types.pizza_type_id = pizzas.pizza_type_id
             ORDER BY 2 DESC
             LIMIT 1;")
q3
# name = The Greek Pizza, price = 35.95
```

### Q#4: Identify the most common pizza size ordered.

```r
q4 <- sqldf("SELECT pizzas.size, COUNT(order_details.order_details_id) AS order_count
             FROM pizzas
             JOIN order_details ON pizzas.pizza_id = order_details.pizza_id
             GROUP BY 1
             ORDER BY 2 DESC
             LIMIT 1;")
q4
# size = L, order_count = 18526
```

### Q#5: Top 5 Most Ordered Pizza Types

```r
q5 <- sqldf("SELECT pizza_types.name, COUNT(order_details.quantity) AS quantity
             FROM pizza_types
             JOIN pizzas ON pizza_types.pizza_type_id = pizzas.pizza_type_id
             JOIN order_details ON order_details.pizza_id = pizzas.pizza_id
             GROUP BY 1
             ORDER BY 2 DESC
             LIMIT 5;")
q5
```

### Q#6: Quantity of Each Pizza Category Ordered

```r
q6 <- sqldf("SELECT pizza_types.category, SUM(order_details.quantity) AS quantity
             FROM pizza_types
             JOIN pizzas ON pizza_types.pizza_type_id = pizzas.pizza_type_id
             JOIN order_details ON order_details.pizza_id = pizzas.pizza_id
             GROUP BY 1
             ORDER BY 2 DESC;")
q6
```

### Q#7: Category-Wise Distribution of Pizzas

```r
q7 <- sqldf("SELECT category AS category_name, COUNT(name) AS pizza_count
             FROM pizza_types
             GROUP BY 1
             ORDER BY 2 DESC;")
q7
```

### Q#8: Category-Wise Distribution of Pizzas

```r
q8 <- sqldf("SELECT category AS category_name, COUNT(name) AS pizza_count
             FROM pizza_types
             GROUP BY 1
             ORDER BY 2 DESC;")
q8
```

### Q#9: Average Number of Pizzas Ordered per Day

```r
q9 <- sqldf("SELECT round(avg(quantity), 0) as avg_pizza_ordered_per_day 
             FROM 
             (SELECT orders.date, SUM(order_details.quantity) AS quantity
              FROM orders
              JOIN order_details ON orders.order_id = order_details.order_id
              GROUP BY 1) AS order_quantity;")
q9
```

### Q#10: Top 3 Most Ordered Pizza Types Based on Revenue

```r
q10 <- sqldf("SELECT pizza_types.name AS pizza_name,
                     SUM(order_details.quantity * pizzas.price) AS revenue
              FROM pizza_types
              JOIN pizzas ON pizzas.pizza_type_id = pizza_types.pizza_type_id
              JOIN order_details ON order_details.pizza_id = pizzas.pizza_id
              GROUP BY 1
              ORDER BY 2 DESC
              LIMIT 3;")
q10
```

### Q#11: Percentage Contribution of Each Pizza Type to Total Revenue

```r
q11 <- sqldf("WITH total_revenue_cte AS (
                    SELECT ROUND(SUM(od.quantity * p.price), 2) AS total_revenue
                    FROM order_details od
                    JOIN pizzas p ON p.pizza_id = od.pizza_id
                 ),
                 category_revenue_cte AS (
                    SELECT pt.category AS pizza_category,
                           SUM(od.quantity * p.price) AS category_revenue
                    FROM pizza_types pt
                    JOIN pizzas p ON p.pizza_type_id = pt.pizza_type_id
                    JOIN order_details od ON od.pizza_id = p.pizza_id
                    GROUP BY pt.category
                 )
                 SELECT cr.pizza_category,
                        ROUND((cr.category_revenue / tr.total_revenue) * 100, 2) AS revenue
                 FROM category_revenue_cte cr
                 CROSS JOIN total_revenue_cte tr
                 ORDER BY revenue DESC;")
q11
```

### Q#12: Cumulative Revenue Over Time

```r
q12 <- sqldf("SELECT date, SUM(revenue) OVER(ORDER BY date) AS cumulative_revenue
              FROM (
                SELECT orders.date,
                       SUM(order_details.quantity * pizzas.price) AS revenue
                FROM order_details
                JOIN pizzas ON order_details.pizza_id = pizzas.pizza_id
                JOIN orders ON orders.order_id = order_details.order_id
                GROUP BY 1) AS sales;")
q12
```

### Q#13: Top 3 Most Ordered Pizza Types by Revenue for Each Category

```r
q13 <- sqldf("SELECT name, revenue
              FROM (
                  SELECT category, name, revenue,
                         RANK() OVER(PARTITION BY category ORDER BY revenue DESC) AS rn
                  FROM (
                      SELECT pizza_types.category, pizza_types.name, 
                             SUM(order_details.quantity * pizzas.price) AS revenue
                      FROM pizza_types
                      JOIN pizzas ON pizza_types.pizza_type_id = pizzas.pizza_type_id
                      JOIN order_details ON order_details.pizza_id = pizzas.pizza_id
                      GROUP BY pizza_types.category, pizza_types.name) AS a ) AS b
              WHERE rn <= 3 LIMIT 3;")
q13
```

---

These comprehensive analyses will provide valuable insights into the performance of a pizzeria. By understanding customer preferences, revenue patterns, and order distributions, businesses can make informed decisions to enhance their operations, marketing strategies, and overall profitability. Data-driven insights like these are essential for sustaining growth and achieving business success in a competitive market.
