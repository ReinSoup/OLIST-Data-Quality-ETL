# Olist Data Quality Pipeline

A Python and pandas data-cleaning project built around the Brazilian E-Commerce Public Dataset by Olist.

I worked through the dataset as a complete data-quality pipeline: inspecting the raw data, assessing quality issues, applying documented cleaning decisions, validating the cleaned outputs, and creating and validating a SQLite database.

## Project Goal

The goal was not to remove as much "messy" data as possible. I first established what each table represented, investigated potential quality issues, and then changed only what I could justify.

The workflow was:

```text
Raw Data
   ->
Inspection
   ->
Quality Assessment
   ->
Cleaning
   ->
Validation
   ->
Cleaned Data
   ->
SQLite Database
   ->
Database Validation
```

## Dataset

I used the Brazilian E-Commerce Public Dataset by Olist, containing approximately 100,000 orders from 2016–2018 across nine related datasets.

Source: [Kaggle - Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

## Tools

- Python
- pandas
- SQLite
- Jupyter Notebook
- Git / GitHub

## Repository Structure

```text
OLIST-Data-Quality-Pipeline/
│
├── data/
│   ├── raw/
│   ├── cleaned/
│   └── database/
│
├── notebooks/
│   ├── 01_Raw_Data_Inspection.ipynb
│   ├── 02_Data_Quality_Assessment.ipynb
│   ├── 03_Data_Cleaning.ipynb
│   ├── 04_Validation.ipynb
│   └── 05_Database_Creation.ipynb
│
├── src/
├── requirements.txt
└── README.md
```

# Project Workflow

## 1. Raw Data Inspection

[Raw Data Inspection](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/01_Raw_Data_Inspection.ipynb)

I inspected all nine source tables before modifying the data.

The inspection focused on:

- Dataset dimensions
- Column names and data types
- Missing values
- Duplicate records
- Identifier uniqueness
- Key relationships between tables
- Suspicious values and inconsistencies

The main purpose was to understand the structure and meaning of the data before deciding what, if anything, needed to change.

### Key findings

- `customers`: no missing values; `customer_id` is unique; customer location data contained potential ZIP/city inconsistencies.
- `orders`: missing timestamps were mostly associated with non-delivered orders, with 8 delivered orders missing the customer delivery date.
- `order_items`: `order_id` legitimately repeats because orders can contain multiple items; `order_id + order_item_id` is the meaningful composite key.
- `order_payments`: no missing values; multiple payment records can belong to one order; 2 credit-card records had zero installments with nonzero payment values.
- `order_reviews`: missing comments were present; multiple reviews can belong to one order; repeated `review_id` values required investigation rather than automatic deletion.
- `products`: 610 products were missing category and several descriptive fields; 2 products were missing all physical measurements; two column names contained spelling errors.
- `sellers`: no missing values; potential ZIP/state inconsistencies were identified.
- `geolocation`: 261,831 exact duplicate rows were found; coordinate anomalies were also screened.
- `product_category_name_translation`: 71 categories were present, with two unique categories lacking English translations.

This inspection provided the evidence for the next stage rather than treating every unusual value as an error.

---

## 2. Data Quality Assessment

[Data Quality Assessment](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/02_Data_Quality_Assessment.ipynb)

I translated the inspection findings into explicit quality decisions before changing the data.

The main principle was:

> An unusual value is not automatically a bad value.

I considered whether an issue was:

1. A clear data-quality problem that could be corrected safely.
2. An ambiguity that should be investigated but preserved.
3. A legitimate structural characteristic of the dataset.
4. An anomaly that could not be corrected reliably without an authoritative source.

### Main assessment decisions

| Finding | Decision | Reason |
|---|---|---|
| Exact duplicate geolocation rows | Remove | Exact copies contain no additional information |
| Order timestamp strings | Convert | Dates should be represented as datetime values |
| Review timestamp strings | Convert | Dates should be represented as datetime values |
| Misspelled product columns | Rename | Correct schema naming without changing the data |
| 8 delivered orders missing delivery date | Preserve | The correct dates could not be reconstructed reliably |
| Repeated review IDs | Preserve | Removing by `review_id` could discard order relationships |
| 2 zero-installment credit-card records | Preserve | The meaning of zero installments was unclear |
| Missing product categories | Preserve | No reliable category could be inferred |
| Missing product dimensions | Preserve | No reliable measurements could be inferred |
| Geographic inconsistencies | Preserve | No authoritative correction was available |
| Untranslated categories | Preserve | I did not invent translations |

The assessment stage established the rules that the cleaning notebook would apply.

---

## 3. Data Cleaning

[Data Cleaning](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/03_Data_Cleaning.ipynb)

I applied the decisions from the quality-assessment stage to the raw DataFrames.

### Cleaning performed

#### Geolocation duplicates

I removed exact duplicate rows:

```python
geolocation = geolocation.drop_duplicates()
```

This reduced the table from 1,000,163 rows to 738,332 rows.

I did not remove repeated ZIP prefixes because multiple geographic records for a ZIP prefix are part of the source data structure.

#### Order timestamps

I converted the order timestamp columns using:

```python
orders[date_columns] = orders[date_columns].apply(pd.to_datetime)
```

Missing timestamps were retained where the source did not provide them.

#### Product column names

I corrected the two spelling errors:

```text
product_name_lenght
-> product_name_length

product_description_lenght
-> product_description_length
```

#### Review timestamps

I converted the review date columns to datetime.

#### Cleaned data export

I exported the cleaned DataFrames to `data/cleaned/` using `to_csv(..., index=False)`.

I kept the raw and cleaned data separate so that the original source data remained untouched.

### What I deliberately did not clean

I did not fabricate delivery dates, product categories, product measurements, translations, geographic values, or payment information.

I also did not remove repeated review IDs simply because they looked duplicated.

This was an important part of the cleaning process: preserving uncertain information is often safer than making an unsupported correction.

---

## 4. Data Validation

[Validation](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/04_Validation.ipynb)

I reloaded the cleaned CSV files and checked whether the intended transformations were actually present.

Validation included:

- Confirming exact geolocation duplicates were removed.
- Confirming the geolocation table contains 738,332 rows.
- Confirming the corrected product column names exist.
- Confirming the old misspellings are gone.
- Converting date columns again to verify they are valid datetime values.
- Confirming the expected missing-date counts remain.
- Confirming the 8 delivered orders with missing customer delivery dates remain.
- Confirming repeated review IDs were preserved.
- Confirming the two zero-installment credit-card records remain.
- Confirming the 610 missing product categories remain.
- Confirming the two products missing all physical dimensions remain.
- Confirming `orders.order_id` remains unique.
- Confirming `products.product_id` remains unique.
- Confirming `sellers.seller_id` remains unique.
- Confirming there are no duplicate `order_id + order_item_id` combinations in `order_items`.

### Important validation lesson

CSV files do not preserve pandas DataFrame dtypes.

Therefore, date columns become strings again when CSV files are reloaded unless they are parsed explicitly.

Validation therefore checked both the values and whether the cleaned data could be interpreted correctly.

---

## 5. SQLite Database

[Database Creation](https://github.com/ReinSoup/OLIST-Data-Quality-Pipeline/blob/main/notebooks/05_Database_Creation.ipynb)

I created a SQLite database from the cleaned datasets using Python's `sqlite3` module and pandas.

Database:

```text
data/database/olist.db
```

I loaded each cleaned DataFrame into its own SQLite table using `DataFrame.to_sql()`.

The database contains nine tables:

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

### Database validation

I verified that:

- All nine tables were created.
- Table row counts match the cleaned datasets.
- Expected columns are present.
- Key identifiers remain valid.
- The `order_id + order_item_id` relationship remains valid.
- Cleaning decisions were preserved in the database.
- SQL queries execute successfully.
- Date columns are represented as `TIMESTAMP` in the SQLite schema.

I used SQLite's `PRAGMA table_info()` to inspect table structures and `COUNT(*)` to verify row counts.

---

# Key Data Quality Findings

The most significant quality findings were:

### 1. Geolocation duplicates

261,831 exact duplicate rows were identified and removed.

The cleaned table contains 738,332 rows.

### 2. Missing delivery dates

8 orders were marked as `delivered` but had no customer delivery date.

I preserved these records because there was no reliable way to reconstruct the missing dates.

### 3. Repeated review IDs

789 `review_id` values appeared more than once across 1,603 rows.

I preserved them because the repeated IDs were associated with different orders, and removing duplicates by `review_id` alone could discard relationships.

### 4. Missing product information

610 products were missing category and several descriptive fields.

2 products were missing all four physical measurements.

I preserved these missing values rather than inventing information.

### 5. Geographic inconsistencies

Potential ZIP, city, state, and coordinate inconsistencies were identified.

I treated these as anomalies requiring authoritative verification rather than automatically correcting them.

---

# Relational Data Understanding

One of the most important parts of the project was learning to interpret the relationships between tables.

For example:

```text
orders
   │
   ├── order_items
   ├── order_payments
   └── order_reviews
```

These are not all one-to-one relationships.

An order can have:

- Multiple items
- Multiple payment records
- Multiple reviews

Therefore, repeated `order_id` values in child tables are often expected.

For `order_items`, the meaningful key is:

```text
order_id + order_item_id
```

For reviews, I learned that a direct merge can legitimately produce repeated order rows when an order has multiple reviews.

This prevented me from confusing legitimate relational structure with duplicate data.

---

# Key Technical Skills

Through the project I practiced:

- Loading multiple CSV files with pandas
- Using `glob` to discover files
- Inspecting DataFrames systematically
- Checking missing values and duplicates
- Testing identifier uniqueness
- Checking relationships between datasets
- Using `.merge()`, `.groupby()`, `.isin()`, and boolean filtering
- Removing exact duplicates with `.drop_duplicates()`
- Renaming columns with `.rename()`
- Converting strings to datetime with `pd.to_datetime()`
- Exporting cleaned CSV files with `.to_csv()`
- Working with SQLite through `sqlite3`
- Loading DataFrames into SQL tables with `.to_sql()`
- Querying SQLite with `pd.read_sql()`
- Inspecting schemas with `PRAGMA table_info()`
- Validating row counts with SQL `COUNT(*)`
- Reasoning about one-to-many relationships and composite keys

I focused on using straightforward, readable Python rather than introducing unnecessary abstraction.

---

# Project Status

| Stage | Status |
|---|---|
| Raw data inspection | Complete |
| Data quality assessment | Complete |
| Data cleaning | Complete |
| Cleaned data export | Complete |
| Data validation | Complete |
| SQLite database creation | Complete |
| SQLite validation | Complete |

The data-quality pipeline is complete through SQLite database creation and validation.

Analysis, dashboards, and machine-learning modeling are outside the current scope and are not claimed as completed stages of this project.

---

# Final Outcome

I started with nine related raw Olist CSV files and built a documented data-quality pipeline that:

1. Inspected the source data.
2. Identified and measured quality issues.
3. Assessed whether each issue required action.
4. Applied only defensible cleaning operations.
5. Preserved ambiguous information instead of inventing corrections.
6. Exported cleaned datasets.
7. Independently validated the cleaned outputs.
8. Built a SQLite database from the cleaned data.
9. Validated the database structure and contents.

The final result is a reproducible, documented workflow from raw e-commerce data to validated cleaned datasets and a queryable SQLite database.
