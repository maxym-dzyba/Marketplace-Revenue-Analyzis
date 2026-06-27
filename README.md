# Marketplace-Revenue-Analyzis

# Business Problem
Why revenue of marketplace doesn't grow for last 5 months?

## Data Overview
- olist_customers_dataset - customers table
- olist_geolocation_dataset - geolocation of customers table
- olist_order_items_dataset - items in orders table
- olist_order_payments_dataset - payments table
- olist_order_reviews_dataset - orders review table
- olist_orders_dataset - orders table
- olist_products_dataset - info about product table
- olist_sellers_dataset - sellers table
- product_category_name_translation - categoory name translation in English table

<img width="2486" height="1496" alt="HRhd2Y0" src="https://github.com/user-attachments/assets/cf29482e-8f64-456f-a0a3-aff9892b4a20" />


## Data Cleaning
- The amount of rows in customer table equals the amount of rows of orders table. If one customer makes 3 orders, there will be 3 duplicates of the same customer in customer table. Decision is not to clean duplicates, but remember about this.
- In olist_order_reviews_dataset there are 138 rows with NULL-value of all column. Decision is to delete them.
- A lot of the records in olist_order_reviews_dataset have bad formatting.  **_Reccomendation** is to understand the reason of it and find a solution how ti fix it_
- 610 products in olist_products_dataset table has no info _Reccomendation is to fill the info about products_
- 2 products category (pc_gamer and portateis_cozinha_e_preparadores_de_alimentos) has not translations in product_category_name_translation table. Decision - insert 2 rows in table manually

## Analysis

### Problem overview

```
WITH year_month_purchase_table AS(
SELECT *,
	SUBSTR(order_purchase_timestamp, 1, 7) AS year_month_purchase
FROM olist_orders_dataset
),
revenue_table AS(
SELECT
ympt.year_month_purchase,
ROUND(SUM(ooid.price), 2) AS revenue,
COUNT(DISTINCT ocd.customer_unique_id) AS customer_count,
COUNT(ooid.order_id) AS item_purchased_count
FROM year_month_purchase_table ympt INNER JOIN olist_order_items_dataset ooid ON ympt.order_id = ooid.order_id
INNER JOIN olist_customers_dataset ocd  ON ympt.customer_id = ocd.customer_id
WHERE ympt.order_status = 'delivered'
GROUP BY ympt.year_month_purchase
ORDER BY  year_month_purchase DESC
),
prev_months_values AS (
SELECT
	year_month_purchase,
	revenue,
	LAG(revenue, 1) OVER(ORDER BY year_month_purchase) AS prev_month_revenue,
	customer_count,
	LAG(customer_count, 1) OVER(ORDER BY year_month_purchase) AS prev_custom_count,
	item_purchased_count,
	LAG(item_purchased_count, 1) OVER(ORDER BY year_month_purchase) AS prev_item_purchased_count
FROM revenue_table
)
SELECT
	year_month_purchase,
	revenue,
	ROUND((revenue - prev_month_revenue) * 100.0 / NULLIF(prev_month_revenue, 0), 2) AS pct_change,
	customer_count,
	ROUND((customer_count - prev_custom_count) * 100.0 / NULLIF(prev_custom_count, 0), 2) AS pct_change,
	item_purchased_count,
	ROUND((item_purchased_count - prev_item_purchased_count) * 100.0 / NULLIF(prev_item_purchased_count, 0), 2) AS pct_change
FROM prev_months_values
ORDER BY year_month_purchase DESC
LIMIT 19
```

On the next screenshot the monthly marketplace revenue, number of customers and number of purchased items is showed:

<img width="929" height="562" alt="image" src="https://github.com/user-attachments/assets/255fbea9-a666-447f-abd5-fb7136652d72" />

All indicators(revenue, number of customers and number of purchased items) changed for last 2 years and showed the growth in common, but for last 5 months (for 7 months for number of customers and number of purchased items) they show non-progress results and even minus. Let's go deep into the data!

### Customers analyzis

As we saw on the screenshot, amount of new customers doesn't grow for last 7 months. During the analyze of repeated orders (customers, who made more than 2 orders) we also see the grow of repeated orders from 2017-01 to 2018-02 and then non-predictable behaviour was started where is mostly falls of repeated orders per month

Orders per value coefficient is average **_1.0139_** for all history, which means that marketplace make focus on the new clients, but doesn't try to encuourage old clients for repeated purchases. Even with it for last 6 months there is the lowest values per history (_**less than 1.01**_) for orders per customer

Monthly average order value shows stable results (_**between 125-140**_ in local currency), which means, that AOV is not the reason of revenue falling down

### Retention

In order to analyze retention, let's make a cohort retention table to see the percentage of customers who make repeated purchases in next months:

```
CREATE TEMP TABLE orders_clean AS -- Temp table was created due to long processing time
SELECT
  ood.order_id,
  ocd.customer_unique_id,
  STRFTIME('%Y-%m', ood.order_purchase_timestamp) AS order_date
FROM olist_orders_dataset ood
JOIN olist_customers_dataset ocd 
  ON ood.customer_id = ocd.customer_id
WHERE ood.order_status = 'delivered';

CREATE TEMP TABLE first_order AS -- Temp table was created due to long processing time
SELECT
  customer_unique_id,
  MIN(order_date) AS first_order
FROM orders_clean
GROUP BY customer_unique_id;

WITH cohort_base AS (
  SELECT
    f.first_order,
    (
      (CAST(SUBSTR(o.order_date, 1, 4) AS INTEGER) - CAST(SUBSTR(f.first_order, 1, 4) AS INTEGER)) * 12 +
      (CAST(SUBSTR(o.order_date, 6, 2) AS INTEGER) - CAST(SUBSTR(f.first_order, 6, 2) AS INTEGER))
    ) AS diff_months,
    COUNT(DISTINCT o.customer_unique_id) AS users_count
  FROM orders_clean o
  JOIN first_order f 
    ON o.customer_unique_id = f.customer_unique_id
  GROUP BY f.first_order, diff_months
),
cohort_size AS (
  SELECT 
    first_order,
    users_count AS cohort_users
  FROM cohort_base
  WHERE diff_months = 0
)
SELECT
  cb.first_order,
  cb.diff_months,
  cb.users_count,
  cs.cohort_users,
  ROUND(1.0 * cb.users_count / cs.cohort_users * 100, 2) AS retention_pct
FROM cohort_base cb
JOIN cohort_size cs
  ON cb.first_order = cs.first_order
ORDER BY cb.first_order, cb.diff_months;
```

Business doesn't make focus on current customers, but only for new customers. Maximum monthly retention percantage is _**0.72%.**_. Value for last 5 months has similar valuer as all previous months. _Reccomendation is to understand does business can spend resources to keep customers to build Life Time Value and increase revenue by this way._

### Categories of products

The most popular categories by items sold amount, orders count, customers count and revenue are:

- Auto, health_beauty, housewares, watches_gifts - shows us grow of all indicators during all history
- bed_bath_table, computers_accessories, furniture_decor, sports_leisure, telephony, toys - shows us no-grow attitude and strong falling down for some categories for last 5-7 months

### Shipping

From 2016-09 to 2017-10 there is _**4.41%**_ average of orders, that was delivered late. But 2017-11 it was _**14.01%**_ then 2018-02 - _**15.74%**_ and finally 2018-03 - _**21.13%**_ which is record. It may have an impact for customers refuse from the orders or just forgot about them.

In deep analyze of this problem It was noticed:

2018-02: 
- 6555 orders, 
- 1032 late orders(15.74% from total), 
- 18 orders took too long to put from status purchased to approved (more than 4 days; 1.74% from all late orders),
- 722 orders took too long to put from status approved to delivered to carrier (As google says, big marketplaces usually do this in a couple of hours, so in our case more than 1 day is long time; 69.96% from all late orders),
- 438 orders took too long to deliver from seller to customer (more than 31 days; 42.44% from all late orders)

2018-03:
- 7003 orders,
- 1480 late (21,13% from total),
- 11 orders purchased-approved (0.74%),
- 992 orders approved-delivered to carrier (67.03%),
- 493 seller-customer (33.31%)

_*Reccomendation is to speed up delivery process, especially the time of giving to seller the order*_

After those 2018-03 the percentage of late orders became better with 5.74%, but it could definitely have impact for future revenue






- аналіз базується тільки на delivered orders
- немає даних про маркетинг і канали залучення
- не враховано зовнішні фактори попиту
