# Retail Sales Analytics Pipeline using Snowflake + AWS S3

## Project Overview

This project demonstrates an end-to-end cloud-based data engineering pipeline using AWS S3 and Snowflake. Raw retail data files are stored in Amazon S3, securely accessed by Snowflake through a Storage Integration and IAM Role, loaded into Snowflake tables, and transformed into analytical datasets for reporting.

## Architecture

```text
Local CSV Files
      ↓
AWS S3 Bucket
      ↓
IAM Role
      ↓
Snowflake Storage Integration
      ↓
External Stage
      ↓
Raw Tables
      ↓
Analytics Tables
```

## Tech Stack

* AWS S3
* AWS IAM
* Snowflake
* SQL
* CSV Data Files

---

# Step 1: Create AWS S3 Bucket

Create an S3 bucket:

```text
snowflake-retail-sales-raw-data-project-bucket
```

Create the following folders:

```text
customers/
products/
orders/
```

Upload the corresponding CSV files into each folder.

---

# Step 2: Create Snowflake Objects

## Create Warehouse

```sql
CREATE WAREHOUSE retail_wh
WITH WAREHOUSE_SIZE = 'XSMALL'
AUTO_SUSPEND = 60
AUTO_RESUME = TRUE;
```

## Create Database

```sql
CREATE DATABASE retail_db;
```

## Create Schema

```sql
CREATE SCHEMA retail_schema;
```

## Set Context

```sql
USE WAREHOUSE retail_wh;
USE DATABASE retail_db;
USE SCHEMA retail_schema;
```

---

# Step 3: Create Raw Tables

## Customers Table

```sql
CREATE OR REPLACE TABLE customers (
    customer_id INT,
    customer_name STRING,
    city STRING,
    state STRING
);
```

## Products Table

```sql
CREATE OR REPLACE TABLE products (
    product_id INT,
    product_name STRING,
    category STRING,
    price NUMBER(10,2)
);
```

## Orders Table

```sql
CREATE OR REPLACE TABLE orders (
    order_id INT,
    customer_id INT,
    product_id INT,
    quantity INT,
    order_date DATE
);
```

---

# Step 4: Create File Format

```sql
CREATE OR REPLACE FILE FORMAT csv_format
TYPE = CSV
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

---

# Step 5: Create IAM Role in AWS

Create an IAM Role named:

```text
SnowflakeS3Role
```

Attach the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSnowflakeS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::snowflake-retail-sales-raw-data-project-bucket",
        "arn:aws:s3:::snowflake-retail-sales-raw-data-project-bucket/*"
      ]
    }
  ]
}
```

---

# Step 6: Create Storage Integration

```sql
CREATE OR REPLACE STORAGE INTEGRATION s3_int
TYPE = EXTERNAL_STAGE
STORAGE_PROVIDER = S3
ENABLED = TRUE
STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<AWS_ACCOUNT_ID>:role/SnowflakeS3Role'
STORAGE_ALLOWED_LOCATIONS = ('s3://snowflake-retail-sales-raw-data-project-bucket/');
```

---

# Step 7: Retrieve Snowflake IAM Details

```sql
DESC INTEGRATION s3_int;
```

Copy:

* STORAGE_AWS_IAM_USER_ARN
* STORAGE_AWS_EXTERNAL_ID

---

# Step 8: Configure Trust Relationship

Update IAM Role Trust Relationship using the values obtained from Snowflake.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
        }
      }
    }
  ]
}
```

---

# Step 9: Create External Stage

```sql
CREATE OR REPLACE STAGE retail_stage
URL='s3://snowflake-retail-sales-raw-data-project-bucket/'
STORAGE_INTEGRATION = s3_int
FILE_FORMAT = csv_format;
```

---

# Step 10: Validate Connectivity

```sql
LIST @retail_stage;
```

Expected output:

```text
customers/
products/
orders/
```

---

# Step 11: Load Data into Snowflake

## Customers

```sql
COPY INTO customers
FROM @retail_stage/customers/customers.csv;
```

## Products

```sql
COPY INTO products
FROM @retail_stage/products/products.csv;
```

## Orders

```sql
COPY INTO orders
FROM @retail_stage/orders/orders.csv;
```

---

# Step 12: Validate Loaded Data

```sql
SELECT * FROM customers;
SELECT * FROM products;
SELECT * FROM orders;
```

---

# Step 13: Create Analytics Table

```sql
CREATE OR REPLACE TABLE sales_report AS
SELECT
    o.order_id,
    c.customer_name,
    p.product_name,
    p.category,
    o.quantity,
    p.price,
    o.quantity * p.price AS total_amount,
    o.order_date
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
JOIN products p
    ON o.product_id = p.product_id;
```

---

# Step 14: Business Queries

## Total Revenue

```sql
SELECT SUM(total_amount) AS total_revenue
FROM sales_report;
```

## Revenue by Category

```sql
SELECT
    category,
    SUM(total_amount) AS revenue
FROM sales_report
GROUP BY category;
```

## Top Selling Products

```sql
SELECT
    product_name,
    SUM(quantity) AS total_quantity
FROM sales_report
GROUP BY product_name
ORDER BY total_quantity DESC;
```

---

# Challenges Encountered

### AWS AssumeRole Error

Error:

```text
User is not authorized to perform: sts:AssumeRole
```

Resolution:

* Verified STORAGE_AWS_IAM_USER_ARN
* Verified STORAGE_AWS_EXTERNAL_ID
* Updated IAM Trust Relationship
* Confirmed IAM Role permissions
* Verified S3 bucket access policy

### Region Mismatch

Observed:

```text
AWS S3 Bucket : us-east-1 (N. Virginia)
Snowflake Account : Asia Pacific (Thailand)
```

Cross-region configurations may introduce additional connectivity and latency considerations. For production deployments, using the same AWS region for both Snowflake and S3 is recommended.

---

# Resume Description

Built an end-to-end cloud data pipeline using Snowflake and AWS S3 by implementing secure Storage Integrations, IAM Role-based authentication, automated data ingestion, SQL transformations, and analytical reporting for retail sales data.

---

# Skills Demonstrated

* Snowflake
* AWS S3
* AWS IAM
* Storage Integration
* External Stages
* COPY INTO
* Data Warehousing
* SQL Transformations
* ETL Pipeline Development
* Cloud Data Engineering
