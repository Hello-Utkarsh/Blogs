So this is my first week of Backend Learning Journey and this week......

# Setup

So firstly let's get our postgres database and tables up and running. I'll be using docker, but you can use any cloud postgres storage also(supabase, render etc)

You can find the tables that I'll be using in the [resource folder of my github]()

So firstly let start our postgres container and start a terminal inside our container
``` bash
docker run -d --name week1pg -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=week1pg -p 5432:5432  postgres:latest

docker exec -it week1pg psql -U postgres
```

Next up create database and tables
```
create database olist;

\c olist

-- 1. Customers Table
CREATE TABLE olist_customers (
    customer_id VARCHAR(32) PRIMARY KEY,
    customer_unique_id VARCHAR(32) NOT NULL,
    customer_zip_code_prefix VARCHAR(10),
    customer_city VARCHAR(100),
    customer_state VARCHAR(2)
);

-- 2. Sellers Table
CREATE TABLE olist_sellers (
    seller_id VARCHAR(32) PRIMARY KEY,
    seller_zip_code_prefix VARCHAR(10),
    seller_city VARCHAR(100),
    seller_state VARCHAR(2)
);

-- 3. Products Table
CREATE TABLE olist_products (
    product_id VARCHAR(32) PRIMARY KEY,
    product_category_name VARCHAR(100),
    product_name_lenght INT, 
    product_description_lenght INT,
    product_photos_qty INT,
    product_weight_g INT,
    product_length_cm INT,
    product_height_cm INT,
    product_width_cm INT
);

-- 4. Orders Table
CREATE TABLE olist_orders (
    order_id VARCHAR(32) PRIMARY KEY,
    customer_id VARCHAR(32) REFERENCES olist_customers(customer_id),
    order_status VARCHAR(50),
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP
);

-- 5. Order Items Table
CREATE TABLE olist_order_items (
    order_id VARCHAR(32) REFERENCES olist_orders(order_id),
    order_item_id INT,
    product_id VARCHAR(32) REFERENCES olist_products(product_id),
    seller_id VARCHAR(32) REFERENCES olist_sellers(seller_id),
    shipping_limit_date TIMESTAMP,
    price DECIMAL(10, 2),
    freight_value DECIMAL(10, 2),
    PRIMARY KEY (order_id, order_item_id)
);

-- 6. Order Payments Table
CREATE TABLE olist_order_payments (
    order_id VARCHAR(32) REFERENCES olist_orders(order_id),
    payment_sequential INT,
    payment_type VARCHAR(50),
    payment_installments INT,
    payment_value DECIMAL(10, 2),
    PRIMARY KEY (order_id, payment_sequential)
);

-- 7. Order Reviews Table
CREATE TABLE olist_order_reviews (
    review_id VARCHAR(32),
    order_id VARCHAR(32) REFERENCES olist_orders(order_id),
    review_score INT,
    review_comment_title VARCHAR(255),
    review_comment_message TEXT,
    review_creation_date TIMESTAMP,
    review_answer_timestamp TIMESTAMP,
    PRIMARY KEY (review_id, order_id)
    
exit
```

Now that we have created our tables, we'll copy the csv files from our local machine into docker container and then copy the data of each file into their respective tables

```
docker cp ./resources/. week1pg:/tmp/olist
docker exec -it week1pg psql -U postgres

COPY olist_customers
FROM '/tmp/olist/olist_customers_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_orders
FROM '/tmp/olist/olist_orders_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_order_items
FROM '/tmp/olist/olist_order_items_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_order_payments
FROM '/tmp/olist/olist_order_payments_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_order_reviews
FROM '/tmp/olist/olist_order_reviews_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_products
FROM '/tmp/olist/olist_products_dataset.csv'
DELIMITER ','
CSV HEADER;

COPY olist_sellers
FROM '/tmp/olist/olist_sellers_dataset.csv'
DELIMITER ','
CSV HEADER;

%% Check whether all the data was transfered form csv file to your tabel using this below commands %%

SELECT COUNT(*) FROM olist_customers;
SELECT COUNT(*) FROM olist_orders;
SELECT COUNT(*) FROM olist_order_items;
SELECT COUNT(*) FROM olist_order_payments;
SELECT COUNT(*) FROM olist_order_reviews;
SELECT COUNT(*) FROM olist_products;
SELECT COUNT(*) FROM olist_sellers;
```

# Query Plans

So suppose you are going on a trip, what's the first thing you do?? Analyze the budget right, we plan everything, how much the room might cost, the food, travel and other expenses, this helps us make our trip according to our budget. Similarly postgres provide `EXPLAIN` and `EXPLAIN ANALYZE`, these two commands helps us track our postgres commands budget and costs(time and resources) that the scripts will costs. It helps us analyze which queries are slow, and taking lot of computing resources

A query plan can be thought of as a tree structure, where each node represents an operation performed by PostgreSQL. The plan describes each step the database takes to complete a query, beginning from the innermost operations (leaf nodes), to the final result (root node).  To understand them effectively, you read them from the "bottom up" or "inside out":

- **Leaf Nodes (Innermost Steps):** These are the base operations that get the raw data. Look for how the data is accessed (e.g., Index Scan or Full Table Scan).
- **Intermediate Nodes (Middle Steps):** These nodes represent operations like filtering, sorting, or aggregating.
- **Root Nodes (Topmost Steps):** The final operation that outputs the result set to the user

## Common Plan Components

When evaluating a query plans you will frequently encounter these critical operation:

- **Access Methods(Scan)**:
	- _Sequential Scan (Seq Scan):_ Reads the entire table. Fast for small tables, but inefficient for large tables without an index.
	- *Parallel Seq Scan*: For large tables, PostgreSQL can run multiple sequential scans simultaneously (in parallel) across CPU cores to reduce scan time.
	- _Index Scan:_ Uses a database index to locate specific rows.
	- *Bitmap Index Scan*: Instead of fetching data rows directly, the database scans the index (like a B-tree) and finds all the matching records. It stores these locations in memory as a **bitmap**—a map of bits where each bit represents a specific row or data page.
	- *Bitmap Heap Scan*: This step takes the bitmap generated by the index scan and uses it to fetch the actual row data from the physical table (the heap).
- **Join Operations**:
	- _Nested Loop:_ Compares rows from one table against rows from another table.
	- _Hash Join:_ Builds a hash table in memory from one table and probes it with another. Highly efficient for large, unsorted datasets.
	- _Merge Join:_ Sorts both tables and merges them.
- **Cost & Cardinality:** Most plans show estimates for _Cost_ (a relative measure of resource consumption) and _Rows_ (how many records the optimizer expects an operation to output).
- **Actual Time**: When you use *EXPLAIN ANALYZE*, you get the real execution time for each node in this form `actual time=20.059..20.063`, here:
	- The the first number(20.059) shows how much time it took to to return the **very first row** or to initialize the node. It is also called **Startup Time**
	- The second number(20.063) is the time taken to process the node entirely and return **all matching rows**. It is also called **Total Time**

### Example Query

```
explain analyze select count(*) from olist_order_items where shipping_limit_date >= '2018-01-01 00:00:00';

                            QUERY PLAN
---------------------------------------------------------------------------
 Aggregate  (cost=3884.12..3884.13 rows=1 width=8) (actual time=24.357..24.360 rows=1.00 loops=1)
   Buffers: shared hit=2320
   ->  Seq Scan on olist_order_items  (cost=0.00..3728.12 rows=62397 width=0) (actual time=0.023..18.610 rows=62515.00 loops=1)
         Filter: (shipping_limit_date >= '2018-01-01 00:00:00'::timestamp without time zone)
         Rows Removed by Filter: 50135
         Buffers: shared hit=2320
 Planning Time: 0.106 ms
 Execution Time: 24.450 ms
(8 rows)
```

You can notice an arrow symbol(`->`), this indicate a child node, in our case that is Seq Scan. So lets start with that:

```
Seq Scan on olist_order_items  (cost=0.00..3728.12 rows=62397 width=0) (actual time=0.023..18.610 rows=62515.00 loops=1)
         Filter: (shipping_limit_date >= '2018-01-01 00:00:00'::timestamp without time zone)
         Rows Removed by Filter: 50135
         Buffers: shared hit=2320
```

- `Buffers: shared hit=2320`: Firstly postgres saves the data in fixed size blocks called *Pages* or *Buffers* which is of 8kb and `shared hit` means that postgres found all the data in the ram(as i've ran this command a few times). It might also show `shared read=2320` which means that postgres had to read through your drive to fetch the data
- `Rows Removed by Filter: 50135`: Since we are filtering the rows using the criteria `shipping_limit_date >= '2018-01-01 00:00:00'`, those removed rows shows the amount of rows that were filtered
- `Seq Scan on olist_order_items  (cost=0.00..3728.12 rows=62397 width=0) (actual time=0.023..18.610 rows=62515.00 loops=1)`: We havent applied any index of the shipping_limit_date that's why postgres had to perform a sequential scan, ie postgres had to scan each row one by one. Also you can notice two brackets with 2 different costs and rows, here the in the 1st one, `cost=...` means that query will cost 0.00 in startup and 3728.12 in executing the query(these are arbitrary unit score for how much CPU/disk work this step will take), `rows=..` means that the planner guessed that the query will return 62397 rows and `width=0` means the width of the data returned per row is 0 bytes (because we only asked for a count, not the actual columns) whereas the 2nd bracket is the actual time in milliseconds (start to finish) for this step, `rows=..` is the exact number of rows that matched our filter and `loop=1` means that the table only needed to scan the table 1 single time to get the answer. 

Now we go above above the tree, after the above explanation I guess understanding the aggregation node is pretty easy right, the same cost, actual time and stuff data, so i would urge you to go through it yourself and try to understand it.



