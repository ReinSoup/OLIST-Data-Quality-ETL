# Olist Data Quality Pipeline

An end-to-end data cleaning and validation project using the Olist e-commerce dataset.

The project demonstrates how raw, inconsistent datasets can be inspected, assessed, cleaned, validated, and transformed into analysis-ready data and a structured SQLite database using Python and pandas.

## Data Source

Brazilian E-Commerce Public Dataset by Olist

Source: [Kaggle, Provided by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

The dataset was provided by Olist and contains approximately 100,000
Brazilian e-commerce orders from 2016 to 2018.

License: CC BY-NC-SA 4.0

## Project Goals

The goal of this project is to build a reproducible data-cleaning workflow rather than simply produce a cleaned dataset.

The workflow covers:

- Raw data inspection
- Data quality assessment
- Missing-value analysis
- Duplicate detection
- Data type correction
- Text and column-name standardization
- Date standardization
- Validation of cleaned data
- Exporting cleaned datasets
- Loading cleaned data into SQLite

## Project Structure

```text
olist-data-quality-pipeline/
│
├── data/
│   ├── raw/
│   │   └── Original Olist datasets
│   │
│   ├── cleaned/
│   │   └── Cleaned and validated datasets
│   │
│   └── database/
│       └── SQLite database
│
├── notebooks/
│   ├── 01_raw_data_inspection.ipynb
│   ├── 02_data_quality_assessment.ipynb
│   └── Data cleaning and validation notebooks
│
├── src/
│   └── Reusable Python cleaning code
│
├── main.py
├── requirements.txt
├── README.md
└── .gitignore
```

## Workflow

```text
Raw Olist Data
      ↓
Data Inspection
      ↓
Quality Assessment
      ↓
Data Cleaning
      ↓
Validation
      ↓
Cleaned Data
      ↓
SQLite Database
```

## Tools Used

- Python
- pandas
- NumPy
- Jupyter Notebook
- SQLite
- SQLAlchemy
- Matplotlib
- Seaborn

## Data

The project uses the Olist Brazilian e-commerce dataset.

The original raw datasets are kept unchanged in `data/raw/`. Cleaning will be performed on working copies so that the original source data remains available for comparison and validation.

The project contains 9 datasets:

1. customers
2. geolocation
3. orders
4. order_items
5. order_payments
6. order_reviews
7. products
8. sellers
9. product_category_name_translation

## Raw Data Inspection

I completed the raw-data inspection across all 9 Olist datasets.

The inspection focused on:

- Dataset dimensions
- Column names and data types
- Missing values
- Duplicate records and keys
- Relationships between datasets
- Value consistency
- Potential data-quality anomalies

Important findings included:

- Missing values in order dates and product information
- Date and timestamp columns stored as strings
- 261,831 exact duplicate rows in the geolocation dataset
- Repeated `review_id` values associated with different `order_id` values
- 8 delivered orders missing `order_delivered_customer_date`
- 2 credit-card payment records with `payment_installments = 0` and nonzero payment values
- Potential geographic inconsistencies in customer, seller, and geolocation data
- 2 misspelled product column names
- 2 product categories without English translations

The inspection also established important relationships between tables. For example, `order_id` represents an order and can legitimately appear multiple times in child datasets such as `order_items`, `order_payments`, and `order_reviews`.

Detailed findings and reasoning are documented in the raw-data inspection notebook.

## Data Quality Assessment

I completed the initial data quality assessment by turning the inspection findings into explicit cleaning decisions.

The main principle I used was:

```text
Find → Measure → Inspect → Decide → Move on
```

I did not treat every unusual or missing value as an error. Where the available data was insufficient to determine the correct value, I chose to preserve the original value rather than make an unsupported assumption.

### Quality Assessment Decisions

| Issue | Finding | Decision | Reason |
|---|---|---|---|
| Geolocation exact duplicates | 261,831 exact duplicate rows | Remove | The duplicate rows contain no additional information |
| Date/timestamp columns | Stored as strings | Convert to datetime | Datetime is the appropriate type for temporal analysis |
| Product column names | `product_name_lenght`, `product_description_lenght` | Rename | Correct the spelling and standardize the column names |
| Delivered orders | 8 delivered orders missing customer delivery date | Preserve | The actual delivery dates cannot be reliably reconstructed |
| Repeated review IDs | 789 review IDs occur multiple times | Preserve | Different `order_id` values may represent meaningful order relationships; removing rows could cause information loss |
| Zero credit-card installments | 2 records have 0 installments with nonzero payment values | Preserve | The meaning of `0` cannot be established reliably from the available data |
| Missing product categories | 610 products have missing categories | Preserve | The actual categories cannot be reliably determined |
| Missing product dimensions | 2 products missing all physical measurements | Preserve | The correct measurements cannot be reconstructed reliably |
| Geographic inconsistencies | Potential ZIP/state and coordinate anomalies | Preserve | No authoritative reference is available for reliable correction |
| Untranslated categories | 2 categories lack English translations | Preserve | I will not invent translations without reliable evidence |

The quality assessment establishes the rules that will be applied during the cleaning phase.

## Project Status

🚧 In progress

Completed:

```text
PHASE 1
Raw Data Inspection ✓
        

      
PHASE 2
Data Quality Assessment ✓
```

Current next stage:

```text
PHASE 3
Data Cleaning ← Next
        
PHASE 4
Validation
        
Cleaned Data Export
        
SQLite Database
        
```

The project does not yet claim that cleaning, validation, cleaned-data export, or SQLite database creation are complete.

## Author

ABDUL RAB (Ryan)
