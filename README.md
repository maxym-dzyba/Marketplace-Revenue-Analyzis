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

<img width="2486" height="1496" alt="HRhd2Y0" src="https://github.com/user-attachments/assets/b66fa38e-8f95-4711-9ed1-7e8871a856aa" />


## Data Cleaning
- The amount of rows in customer table equals the amount of rows of orders table. If one customer makes 3 orders, there will be 3 duplicates of the same customer in customer table. Decision is not to clean duplicates, but remember about this.
- In olist_order_reviews_dataset there are 138 rows with NULL-value of all column. Decision is to delete them.
- A lot of the records in olist_order_reviews_dataset have bad formatting.  _Reccomendation is to understand the reason of it and find a solution how ti fix it_
- 610 products in olist_products_dataset table has no info _Reccomendation is to fill the info about products_

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
COUNT(DISTINCT ocd.customer_id) AS customer_count,
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
<img width="932" height="592" alt="image" src="https://github.com/user-attachments/assets/05b6dbcd-d78a-4a13-b09f-d62d42163b64" />

All indicators(revenue, number of customers and number of purchased items) changed for last 2 years and showed the growth in common, but for last 5 months (for 7 months for number of customers and number of purchased items) they show non-progress results and even minus. Let's go deep into the data!
