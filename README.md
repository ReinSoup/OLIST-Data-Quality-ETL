# Olist Data Quality Pipeline

An end-to-end data cleaning and validation project using the Olist e-commerce dataset.

The project demonstrates how raw, inconsistent datasets can be inspected, cleaned, validated, and transformed into analysis-ready data and a structured SQLite database using Python and pandas.

## Project Goals

The goal of this project is to build a reproducible data-cleaning workflow rather than simply produce a cleaned dataset.

The workflow covers:

- Raw data inspection
- Data quality assessment
- Missing-value analysis
- Duplicate detection
- Data type correction
- Text normalization
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
│   ├── Data inspection
│   ├── Data cleaning
│   └── Data validation
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
Identify Quality Issues
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

The original raw datasets are included in `data/raw/` so that the complete transformation from raw data to cleaned data can be examined.

The raw files are kept unchanged.


## Inspection Conclusion

The raw-data inspection phase is complete across all 9 Olist datasets.

Key findings include:

- Missing values in several datasets, including order dates and product information
- Date and timestamp columns stored as strings
- Exact duplicate records in the geolocation dataset
- Suspicious repeated review IDs
- Inconsistent seller geographic information
- A small number of suspicious geolocation coordinates
- Two misspelled product column names
- A small number of product categories without English translations
- A small number of payment records requiring further investigation

Not every unusual value is an error. I assessed issues in the context of each table's structure and relationships before deciding whether they should be cleaned, investigated further, or left unchanged.

Detailed findings and reasoning are documented in the raw-data inspection notebook rather than duplicated here.

## Project Status

🚧 In progress

Current stage: Raw Data Inspection ✓

The raw-data inspection phase has been completed across all 9 datasets using a two-level inspection process:

1. Inspect the entire collection of datasets at a high level
2. Inspect each CSV individually

The project is now moving into:

```text
PHASE 1
Global inspection ✓
        ↓
PHASE 2
Detailed quality assessment ✓
        ↓
PHASE 3
Cleaning ← Current next stage
        ↓
PHASE 4
Validation
        ↓
Cleaned Data
        ↓
SQLite Database
```


## Author

ABDUL RAB (Ryan)

