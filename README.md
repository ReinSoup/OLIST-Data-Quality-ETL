# Olist Data Quality Pipeline

A Python and pandas data-cleaning project using the Brazilian E-Commerce Public Dataset by Olist.

I built this project as a complete data-quality workflow rather than simply applying cleaning functions to a dataset. I inspected all nine source tables, assessed the findings, made explicit cleaning decisions, exported cleaned data, validated the results, and created and validated a SQLite database.

## Project Overview

The objective of this project was to take a collection of raw e-commerce CSV files and turn them into a cleaner, validated, queryable dataset while documenting why each transformation was or was not performed.

I followed this workflow:

```text
Raw Data, 
   
Raw Data Inspection,
   
Data Quality Assessment,
   
Data Cleaning,
   
Data Validation,
   
Cleaned CSV Export,
   
SQLite Database Creation,
   
SQLite Validation,
```

My guiding principle throughout the project was:

```text
Find -> Measure -> Inspect -> Decide -> Move on
```

This kept the investigation systematic without turning it into an unnecessary search for problems that did not matter.

---

## Dataset

I used the Brazilian E-Commerce Public Dataset by Olist.

Source: Kaggle, provided by Olist  
Dataset: Brazilian E-Commerce Public Dataset by Olist  
Kaggle: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

The dataset contains approximately 100,000 orders from 2016–2018 and is distributed across nine CSV files.

### Source Tables

| Dataset | Purpose |
|---|---|
| `customers` | Customer records and customer location information |
| `geolocation` | ZIP-prefix geographic information |
| `orders` | One record for each order |
| `order_items` | Items contained in each order |
| `order_payments` | Payment records associated with orders |
| `order_reviews` | Customer review records |
| `products` | Product attributes |
| `sellers` | Seller information |
| `product_category_name_translation` | Portuguese to English category mapping |

---

## Project Structure

```text
OLIST_Data_Cleaning/
│
├── data/
│   ├── raw/
│   ├── cleaned/
│   └── database/
│
├── notebooks/
│   ├── 01_raw_data_inspection.ipynb
│   ├── 02_data_quality_assessment.ipynb
│   ├── 03_data_cleaning.ipynb
│   ├── 04_data_validation.ipynb
│   └── 05_sqlite_database.ipynb
│
├── src/
├── requirements.txt
└── README.md
```

The `raw` directory contains the original CSV files.

The `cleaned` directory contains the cleaned CSV exports.

The `database` directory contains the SQLite database created from the cleaned datasets.

---

# 1. Raw Data Inspection

Notebook:
[Raw Data Inspection](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/01_Raw_Data_Inspection.ipynb)

I inspected all nine datasets before modifying any data.

I did not start by blindly filling missing values, dropping duplicates, or changing unusual records. Instead, I first tried to understand what each table represented and what one row meant.

For each dataset, I focused on:

- Shape
- Column names
- Data types
- Missing values
- Duplicate records
- Identifier uniqueness
- Relationships between tables
- Important value distributions
- Suspicious or inconsistent records

This established the evidence used later in the quality-assessment stage.

## Why I separated inspection from cleaning

I treated these as different stages:

```text
Inspection
“What did I find?”

Quality Assessment
“What does each finding mean, and what should I do about it?”

Cleaning
“Apply those decisions.”

Validation
“Prove that the cleaning worked and did not introduce new problems.”
```

This distinction prevented me from treating every unusual value as an error.

---

# 2. Data Quality Assessment

Notebook:

[Data Quality Assessment](notebooks/02_data_quality_assessment.ipynb)

I converted the inspection findings into explicit decisions before changing the data.

The central rule I followed was:

> An unusual value is not automatically a bad value.

If I could establish that a record was an exact duplicate with no additional information, I removed it.

If a value was unusual but its correct replacement could not be established reliably, I preserved it.

This was especially important for geographic anomalies, missing product information, repeated review IDs, and ambiguous payment values.

---

# 3. Assessment by Dataset

## Customers

Shape:

```text
99,441 × 5
```

I found:

- No missing values.
- `customer_id` is unique.
- `customer_unique_id` is not unique.
- Every `customer_id` referenced by `orders` exists in `customers`.
- Every customer record in `orders` is associated with one order.
- Every ZIP prefix maps to one state.
- 39 ZIP prefixes have multiple city names.
- 476 customer records are associated with those prefixes.

### Decision

I treated the city/ZIP relationship as a potential consistency issue rather than automatically correcting it.

I did not have an authoritative geographic source that would justify replacing the supplied city values.

---

## Orders

Shape:

```text
99,441 × 8
```

Missing values:

| Column | Missing |
|---|---:|
| `order_approved_at` | 160 |
| `order_delivered_carrier_date` | 1,783 |
| `order_delivered_customer_date` | 2,965 |

The remaining columns had no missing values.

Order statuses:

| Status | Count |
|---|---:|
| `delivered` | 96,478 |
| `shipped` | 1,107 |
| `canceled` | 625 |
| `unavailable` | 609 |
| `invoiced` | 314 |
| `processing` | 301 |
| `created` | 5 |
| `approved` | 2 |

I found eight orders marked as `delivered` without an `order_delivered_customer_date`.

I also found that missing delivery dates generally correspond to orders that were not delivered.

### Important terminology

I learned that:

```text
order_delivered_carrier_date
```

represents the point at which an order was handed to the carrier.

It does not mean that the customer received the order.

### Decisions

I converted the date columns to datetime during cleaning.

I preserved the eight delivered orders with missing customer-delivery dates because the actual dates could not be reconstructed reliably.

I did not invent delivery dates.

I also confirmed that:

- `order_id` is unique.
- `customer_id` is unique within `orders`.
- All order customer IDs exist in `customers`.

---

## Order Items

Shape:

```text
112,650 × 7
```

I found no missing values.

I found that `order_id` is not unique. This is expected because an order can contain multiple items.

I also learned that `order_item_id` is a sequence within an order rather than a globally unique identifier.

The meaningful key is:

```text
order_id + order_item_id
```

I found no duplicate combinations of this composite key.

I also confirmed that:

- All `order_id` values exist in `orders`.
- All `product_id` values exist in `products`.
- Item sequences are consistent.

### Decision

I found no cleaning issue that justified changing this table.

---

## Order Payments

Shape:

```text
103,886 × 5
```

I found no missing values.

Payment types:

| Payment type | Count |
|---|---:|
| `credit_card` | 76,795 |
| `boleto` | 19,784 |
| `voucher` | 5,775 |
| `debit_card` | 1,529 |
| `not_defined` | 3 |

I found no duplicate `order_id + payment_sequential` combinations.

### Payment sequence

I learned that `payment_sequential` represents the position of a payment record within an order.

It does not represent the number of installments.

An order can therefore have multiple payment records, including multiple voucher payments.

### Payment anomalies

The three `not_defined` payment records have a `payment_value` of zero.

I also found zero-value payments among vouchers, so I did not treat a zero payment value as automatically invalid.

Two credit-card records have:

```text
payment_installments = 0
```

while having nonzero payment values.

### Decision

I preserved the two zero-installment credit-card records because the meaning of the value was unclear.

I did not invent a corrected installment count.

I also confirmed that no negative payment values existed.

---

## Order Reviews

Shape:

```text
99,224 × 7
```

I found missing values in:

```text
review_comment_title
review_comment_message
```

I did not treat missing comments as data-quality errors because a customer can legitimately submit a review without written comments.

Review scores were all within the expected 1–5 range.

All review `order_id` values exist in `orders`.

### Multiple reviews per order

I found:

| Reviews for an order | Orders |
|---:|---:|
| 1 | 98,126 |
| 2 | 543 |
| 3 | 4 |

This taught me an important relational-data concept.

`orders` represents one row per order, while `order_reviews` represents one row per review.

Therefore, a merge on `order_id` can legitimately repeat an order when that order has multiple reviews.

That is not automatically a duplicate-data problem.

If I need one row per order, I can aggregate the reviews first:

```python
review_summary = (
    order_reviews
    .groupby("order_id")["review_score"]
    .mean()
    .reset_index()
)

orders.merge(review_summary, on="order_id", how="left")
```

### Repeated review IDs

I found:

```text
97,621 review IDs appear once
764 review IDs appear twice
25 review IDs appear three times
```

This results in 789 repeated review IDs across 1,603 rows.

The repeated IDs were associated with different order IDs.

### Decision

I preserved the repeated review IDs.

I deliberately did not use:

```python
order_reviews.drop_duplicates("review_id")
```

because that would arbitrarily retain one order relationship and potentially discard information.

This was an important example of why duplicate detection must be based on business meaning rather than simply looking for repeated values in one column.

I converted the review date columns to datetime during cleaning.

---

## Products

Shape:

```text
32,951 × 9
```

I confirmed that `product_id` is unique.

### Column-name corrections

The raw dataset contains these misspellings:

```text
product_name_lenght
product_description_lenght
```

I renamed them to:

```text
product_name_length
product_description_length
```

### Missing values

| Column | Missing |
|---|---:|
| `product_category_name` | 610 |
| `product_name_lenght` | 610 |
| `product_description_lenght` | 610 |
| `product_photos_qty` | 610 |
| `product_weight_g` | 2 |
| `product_length_cm` | 2 |
| `product_height_cm` | 2 |
| `product_width_cm` | 2 |

The same 610 products are missing the category and several descriptive fields.

All 610 missing-category products appear in `order_items`, meaning these are products that actually occur in order data.

Two products are missing all four physical measurements.

### Decisions

I corrected the column-name spelling.

I preserved the missing category values.

I preserved the missing product measurements.

I did not invent categories, weights, or dimensions.

---

## Sellers

Shape:

```text
3,095 × 4
```

I found:

- No missing values.
- `seller_id` is unique.
- All seller IDs referenced by `order_items` exist in `sellers`.

### Geographic consistency

I found:

```text
2,229 ZIP prefixes → one state
16 ZIP prefixes → two states
1 ZIP prefix → three states
```

Some inspected examples showed the same city and ZIP prefix associated with different states.

### Decision

I preserved these geographic inconsistencies.

Without an authoritative geographic reference, changing the values would have been speculation rather than data cleaning.

---

## Geolocation

Shape:

```text
1,000,163 × 5
```

I found:

- No missing values.
- 19,015 unique ZIP prefixes.
- Repeated ZIP prefixes are expected because multiple geographic records can exist for the same ZIP prefix.

### Exact duplicates

I found:

```text
261,831 exact duplicate rows
```

After removing exact duplicate rows:

```text
738,332 rows
```

I considered these duplicates safe to remove because the rows were exact copies and contained no additional information.

I used:

```python
geolocation = geolocation.drop_duplicates()
```

I also learned that:

```python
geolocation.drop_duplicates()
```

returns a new DataFrame and does not modify the original unless I assign the result back or explicitly use an in-place operation.

### Coordinate screening

I screened coordinates against broad Brazil coordinate bounds and found 37 records outside those bounds.

I treated this only as a screening test, not proof that the records were incorrect.

Some inspected records also showed city, state, and coordinate inconsistencies.

### Decision

I removed exact duplicate rows.

I preserved geographic coordinate anomalies because I did not have an authoritative source from which to reconstruct the correct coordinates.

---

## Product Category Translation

Shape:

```text
71 × 2
```

I found:

- No missing values.
- 71 unique categories.
- 13 product rows belong to categories without English translations.

The two unique untranslated categories were:

```text
pc_gamer
portateis_cozinha_e_preparadores_de_alimentos
```

### Decision

I preserved these categories rather than inventing translations.

---

# 4. Cleaning

Notebook:

```text
[Cleaning](notebooks/03_data_cleaning.ipynb)
```

I created the cleaning notebook only after completing the inspection and assessment stages.

The cleaning stage applied the decisions I had already made.

## Loading the raw CSV files

I used Python's `glob` module with pandas to load the CSV files:

```python
import pandas as pd
import glob

csv_files = glob.glob(
    r"C:\Users\Abdul\Documents\VS_code_projects\OLIST_Data_Cleaning\data\raw\*.csv"
)

datasets = {}

for file in csv_files:
    data = pd.read_csv(file)
    datasets[file] = data
```

I then assigned each DataFrame explicitly so that I could work with readable variable names.

I chose this approach because it was simple, readable, and appropriate for the scale of the project.

## Removing exact geolocation duplicates

```python
geolocation = geolocation.drop_duplicates()
```

This reduced the geolocation table from 1,000,163 rows to 738,332 rows.

I did not remove repeated ZIP prefixes because repeated ZIP prefixes are part of the structure of the source data and are not equivalent to exact duplicate rows.

## Converting order dates

```python
date_columns = [
    "order_purchase_timestamp",
    "order_approved_at",
    "order_delivered_carrier_date",
    "order_delivered_customer_date",
    "order_estimated_delivery_date"
]

orders[date_columns] = orders[date_columns].apply(pd.to_datetime)
```

I converted the order timestamp columns from strings to pandas datetime values.

I allowed missing values to remain missing rather than fabricating dates.

## Renaming product columns

```python
products = products.rename(columns={
    "product_name_lenght": "product_name_length",
    "product_description_lenght": "product_description_length"
})
```

I corrected the spelling while preserving the meaning and values of the columns.

## Converting review dates

```python
review_date_columns = [
    "review_creation_date",
    "review_answer_timestamp"
]

order_reviews[review_date_columns] = (
    order_reviews[review_date_columns].apply(pd.to_datetime)
)
```

## Exporting cleaned datasets

I exported the cleaned DataFrames to the `data/cleaned` directory using `to_csv(..., index=False)`.

I exported the cleaned DataFrames themselves rather than stale copies stored in the original dataset dictionary.

I used `index=False` so that the pandas DataFrame index would not become an unwanted CSV column.

I also learned that the default write mode is `"w"`, meaning an existing file is replaced when the cleaned CSV is exported.

---

# 5. Cleaning Decisions

The final cleaning rules were:

| Finding | Action | Reason |
|---|---|---|
| Exact duplicate geolocation rows | Remove | Exact copies contain no additional information |
| Order date strings | Convert | Correct representation for temporal data |
| Review date strings | Convert | Correct representation for temporal data |
| Misspelled product columns | Rename | Correct schema naming without changing values |
| 8 delivered orders missing delivery date | Preserve | Correct date cannot be reconstructed reliably |
| Repeated review IDs | Preserve | Dropping by `review_id` could discard order relationships |
| 2 zero-installment credit-card records | Preserve | Meaning of zero is unclear |
| Missing product categories | Preserve | No reliable category can be inferred |
| Missing product dimensions | Preserve | No reliable measurements can be inferred |
| Seller geographic inconsistencies | Preserve | No authoritative correction available |
| Geolocation coordinate anomalies | Preserve | Bounds were only a screening test |
| Untranslated categories | Preserve | I did not invent translations |
| Customer city/ZIP inconsistencies | Preserve | Potential inconsistency, but not enough evidence for correction |

The most important decision was not what I removed. It was what I deliberately chose not to remove.

---

# 6. Data Validation

Notebook:

```text
[Validation](notebooks/04_data_validation.ipynb)
```

I did not consider the project finished simply because the cleaning code executed without errors.

I reloaded the cleaned CSV files and independently checked whether the intended results were actually present.

## Important CSV behavior

I learned that CSV files do not preserve pandas DataFrame dtypes.

Therefore, when I reloaded the cleaned CSV files, date columns were strings again unless I explicitly parsed them.

During validation I converted the date columns back to datetime in order to test whether the values could be interpreted correctly.

## Validation checks

I confirmed:

- Geolocation exact duplicates = 0.
- Geolocation shape = 738,332 rows.
- The corrected product column names exist.
- The old misspelled product column names are gone.
- Order date columns convert successfully to datetime.
- Order missing-date counts match the original findings.
- Review date columns convert successfully to datetime.
- The eight delivered orders with missing customer-delivery dates are still present.
- The 789 repeated review IDs across 1,603 rows are still present.
- The two zero-installment credit-card records are still present.
- The 610 missing product categories are still present.
- The two products missing all physical dimensions are still present.
- `orders.order_id` remains unique.
- `products.product_id` remains unique.
- `sellers.seller_id` remains unique.
- `order_items` has zero duplicate `order_id + order_item_id` combinations.

This confirmed that the cleaning stage applied the intended changes without accidentally deleting records that I had decided to preserve.

---

# 7. SQLite Database

Notebook:

```text
[SQLITE Database](notebooks/05_sqlite_database.ipynb)
```

I created a SQLite database from the cleaned datasets using Python's `sqlite3` module and pandas.

Database:

```text
data/database/olist.db
```

## Database connection

```python
import pandas as pd
import sqlite3

db_path = r"C:\Users\Abdul\Documents\VS_code_projects\OLIST_Data_Cleaning\data\database\olist.db"

conn = sqlite3.connect(db_path)
```

The connection variable I used was `conn`.

Because the cleaned CSVs were reloaded before database creation, I converted the relevant date columns back to datetime before loading them into SQLite.

## Creating the tables

I loaded all nine cleaned datasets into separate SQLite tables:

```python
customers.to_sql("customers", conn, if_exists="replace", index=False)
geolocation.to_sql("geolocation", conn, if_exists="replace", index=False)
orders.to_sql("orders", conn, if_exists="replace", index=False)
order_items.to_sql("order_items", conn, if_exists="replace", index=False)
order_payments.to_sql("order_payments", conn, if_exists="replace", index=False)
order_reviews.to_sql("order_reviews", conn, if_exists="replace", index=False)
products.to_sql("products", conn, if_exists="replace", index=False)
sellers.to_sql("sellers", conn, if_exists="replace", index=False)
category_translation.to_sql(
    "product_category_name_translation",
    conn,
    if_exists="replace",
    index=False
)
```

I intentionally wrote each table explicitly rather than hiding the process inside a loop. This made the database-creation stage easier for me to understand and inspect.

`if_exists="replace"` replaces an existing table.

`index=False` prevents the pandas index from being stored as an extra database column.

---

# 8. SQLite Schema

The final database contains:

| Table | Rows | Columns |
|---|---:|---:|
| `customers` | 99,441 | 5 |
| `geolocation` | 738,332 | 5 |
| `orders` | 99,441 | 8 |
| `order_items` | 112,650 | 7 |
| `order_payments` | 103,886 | 5 |
| `order_reviews` | 99,224 | 7 |
| `products` | 32,951 | 9 |
| `sellers` | 3,095 | 4 |
| `product_category_name_translation` | 71 | 2 |

The geolocation table has fewer rows than the raw dataset because I removed exact duplicates.

## Inspecting the schema

I used SQLite's `PRAGMA table_info()` command:

```python
query = """
PRAGMA table_info(order_reviews);
"""

df = pd.read_sql(query, conn)
df
```

I used this to inspect column names and declared SQLite types.

The date columns appeared as `TIMESTAMP`.

SQLite is flexible with data types and does not have a strict native datetime storage type in the same sense as some database systems. `TIMESTAMP` is therefore a reasonable declared representation rather than evidence of a strict native timestamp type.

---

# 9. SQL Lessons

## SQL date literals need quotes

I initially tested a date filter like:

```sql
WHERE review_creation_date > 2019-01-18
```

The correct form is:

```sql
WHERE review_creation_date > '2019-01-18'
```

Without quotes, SQLite interprets `2019-01-18` as an expression rather than a quoted date literal.

For example:

```python
query = """
SELECT review_id, review_creation_date
FROM order_reviews
WHERE review_creation_date > '2018-01-18'
"""

df = pd.read_sql(query, conn)
df
```

The Olist review dates are from 2017–2018, so this query should return no rows.

## SQL does not have pandas `.shape`

A pandas DataFrame can use:

```python
df.shape
```

A SQL table does not have that pandas property.

I used:

```sql
SELECT COUNT(*) AS row_count
FROM customers;
```

through pandas:

```python
query = """
SELECT COUNT(*) AS row_count
FROM customers;
"""

pd.read_sql(query, conn)
```

For table structure, I used:

```sql
PRAGMA table_info(customers);
```

This distinction helped me understand the difference between working with DataFrames and working with relational database tables.

## One query at a time

I attempted a combined multi-subquery row-count query.

It was unnecessary for this project and did not work as expected.

I chose simple one-table-at-a-time queries instead because they were easier to understand, verify, and debug.

---

# 10. SQLite Validation

I validated the completed database and confirmed:

- All nine expected tables were created.
- Row counts match the cleaned datasets.
- The geolocation row count reflects exact duplicate removal.
- Expected columns are present.
- Key identifiers remain valid.
- `order_id + order_item_id` remains a valid composite relationship for `order_items`.
- The cleaning decisions were preserved in the database.
- Date columns are represented as `TIMESTAMP`.
- SQL queries execute successfully.

I also verified the expected table sizes:

```text
customers                              99,441
geolocation                           738,332
orders                                 99,441
order_items                           112,650
order_payments                        103,886
order_reviews                          99,224
products                               32,951
sellers                                 3,095
product_category_name_translation         71
```

When the database work was complete, I closed the connection:

```python
conn.close()
```

---

# 11. Relational Data Concepts I Learned

This project taught me that data cleaning is not only about values inside individual cells. Understanding relationships between tables is equally important.

## One-to-many relationships

An order can contain multiple items.

Therefore:

```text
orders
1 order
   ↓
many order_items
```

This is why `order_id` repeats in `order_items`.

Similarly, an order can have multiple review records.

```text
orders
1 order
   ↓
many order_reviews
```

A repeated `order_id` after a merge is therefore not automatically a duplicate.

## Business keys

I learned to distinguish between a column that looks unique and an identifier that is meaningful within the business structure.

For `order_items`, `order_item_id` is not globally unique.

The meaningful key is:

```text
order_id + order_item_id
```

## Merge reasoning

Before joining tables, I learned to ask:

1. What does one row represent in each table?
2. Is the join key unique?
3. What relationship exists between the tables?
4. Should I expect one-to-one, one-to-many, or many-to-many behavior?

Pandas:

```python
orders.merge(order_reviews, on="order_id")
```

is conceptually equivalent to a SQL join:

```sql
SELECT *
FROM orders
JOIN order_reviews
ON orders.order_id = order_reviews.order_id;
```

The structure of the data determines whether repeated rows after the join are expected.

---

# 12. Python and pandas Concepts I Practiced

Throughout the project I learned and applied:

- `pd.read_csv()`
- `glob.glob()`
- Dictionaries containing DataFrames
- DataFrame indexing and assignment
- Boolean filtering
- `.isin()`
- `.groupby()`
- `.merge()`
- `.drop_duplicates()`
- `.rename()`
- `pd.to_datetime()`
- `.to_csv()`
- `index=False`
- `sqlite3.connect()`
- `DataFrame.to_sql()`
- `pd.read_sql()`
- `PRAGMA table_info()`
- SQL `COUNT(*)`
- SQL filtering
- Composite-key reasoning
- One-to-many relationships
- Validation through explicit checks

I focused on the simplest practical method for each task instead of introducing unnecessary abstraction.

---

# 13. Important Lessons From the Project

## Missing data is not automatically bad data

I learned that a missing value has to be interpreted in context.

For example:

- A missing review comment can be legitimate.
- A missing delivery date can be expected for an undelivered order.
- A missing product dimension may simply mean the source did not provide the information.

I therefore avoided blindly filling or deleting missing values.

## Duplicate values are not automatically duplicate records

I learned to distinguish between:

```text
Repeated value
```

and:

```text
Repeated record
```

For example, multiple rows with the same `order_id` in `order_items` are expected.

The geolocation exact duplicates were safe to remove because the entire rows were identical.

The repeated `review_id` values were not safe to remove based on that column alone because doing so could discard order relationships.

## Anomalies require evidence

I found geographic inconsistencies and coordinate anomalies.

I did not automatically correct them.

A screening rule can tell me:

```text
“This record deserves investigation.”
```

It does not necessarily prove:

```text
“This record is wrong.”
```

Without a reliable authoritative source, changing those values would have been speculation.

## Cleaning should be reversible and explainable

I kept the raw data separate from the cleaned data.

The cleaning operations were explicit and documented.

This makes it possible to explain:

- What changed
- Why it changed
- What was intentionally left unchanged

## Validation is part of cleaning

I learned that running cleaning code successfully is not proof that the cleaned dataset is correct.

I therefore reloaded the outputs and checked:

```text
Did the intended changes happen?
Did the intended records remain?
Did keys remain valid?
Did row counts make sense?
Did I accidentally remove information?
```

That validation stage is what gives the cleaning process credibility.

---

# 14. Final Quality Decisions

The final project decisions can be summarized as follows:

```text
REMOVE
────────────────────────────────────
Exact duplicate geolocation rows


CONVERT
────────────────────────────────────
Order timestamp strings → datetime
Review timestamp strings → datetime


RENAME
────────────────────────────────────
product_name_lenght
→ product_name_length

product_description_lenght
→ product_description_length


PRESERVE
────────────────────────────────────
8 delivered orders with missing delivery dates
Repeated review IDs
2 zero-installment credit-card records
Missing product categories
Missing product dimensions
Seller geographic inconsistencies
Geolocation coordinate anomalies
Untranslated product categories
Customer city/ZIP inconsistencies
```

The most important decision was not what I removed. It was what I deliberately chose not to remove.

---

# 15. Project Completion Status

| Stage | Status |
|---|---|
| Raw data inspection | Complete |
| Data quality assessment | Complete |
| Data cleaning | Complete |
| Cleaned CSV export | Complete |
| Data validation | Complete |
| SQLite database creation | Complete |
| SQLite validation | Complete |

The data-quality pipeline is complete through SQLite creation and validation.

I have not claimed analysis, dashboard development, or machine-learning modeling as completed because those are separate stages beyond the current scope of this project.

---

# 16. What This Project Demonstrates

This project demonstrates that I can:

- Work with multiple related CSV datasets.
- Inspect unfamiliar data systematically.
- Identify missing values and duplicate records.
- Reason about identifiers and composite keys.
- Investigate relationships between tables.
- Distinguish legitimate repeated records from actual duplicates.
- Assess data-quality problems before changing the data.
- Apply defensible cleaning rules with pandas.
- Preserve ambiguous data rather than inventing corrections.
- Validate cleaned outputs independently.
- Export cleaned datasets.
- Build a SQLite database from cleaned data.
- Inspect a database schema with SQL.
- Query SQLite through pandas.
- Validate database row counts and structure.
- Document data-quality decisions clearly.

The project is therefore not just a collection of pandas transformations. It is a documented data-quality pipeline from raw data through a validated SQLite database.

---

# 17. Final Outcome

I started with nine raw Olist CSV files and built a complete workflow for:

```text
Raw CSVs
   |
Systematic Inspection
   |
Evidence-Based Quality Assessment
   |
Documented Cleaning
   |
Cleaned CSVs
   |
Independent Validation
   |
SQLite Database
   |
SQL-Based Validation
```

The final database contains all nine cleaned datasets, with the intentional cleaning decisions preserved.

The project demonstrates the full cycle of understanding data before modifying it, making defensible decisions, applying those decisions, and then proving that the resulting data is consistent with the intended outcome.
