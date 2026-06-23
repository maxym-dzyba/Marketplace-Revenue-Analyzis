# Marketplace-Revenue-Analyzis

# Business Problem
Why revenue of marketplace doesn't grow for last 6 months?

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
