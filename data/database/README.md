# Olist Database

This folder contains the SQLite database created from the cleaned Olist e-commerce dataset.

## Purpose

The database stores the cleaned Olist data in related tables, making it easier to query, analyze, and work with the data using SQL.

## Contents

- SQLite database containing the cleaned Olist tables
- Related tables connected through primary and foreign keys
- Data prepared from the cleaning and validation stages of the project

## Database Structure

The database follows the relationships present in the original Olist dataset, including orders, customers, products, sellers, payments, reviews, and related information.

The database is the final structured storage layer of the ETL pipeline.

## Usage

The database can be opened and queried using SQLite, Python, or database tools such as VS Code extensions.

```sql
SELECT *
FROM orders
LIMIT 10;
