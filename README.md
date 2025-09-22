# NYC Restaurant Inspections (Data Cleaning & SQL)
 
**Tech stack:** Python (pandas, sqlite3), SQL (SQLite), Jupyter Notebook  

---

## Summary

This project demonstrates end-to-end data preprocessing and cleaning for NYC restaurant inspection and ratings data.  
Cleaned tables are stored in a SQLite database (`restaurant_data.db`) and analytical SQL queries are run to answer practical questions.  
This README intentionally does **not** include figures; instead, long SQL outputs are exported to CSV files for inspection.

---

## High-level goals

1. Retrieve raw inspection and ratings data (`pd.read_html()` and API).  
2. Clean and standardize both tables (dates, types, names, price levels, ZIP codes, dtypes).  
3. Save cleaned DataFrames into a SQLite database.  
4. Run analytical SQL queries (average scores by cuisine, price-level impact, pass rates by inspection type).  
5. Export long SQL results to CSV files for sharing.

---

## Steps in the notebook

### 1) Load required modules
`import os, pandas as pd, sqlite3, requests`  
`from dotenv import load_dotenv`

### 2) Scrape inspections table
`url = "https://dlai-lc-dag.s3.us-east-2.amazonaws.com/restaurant_inspections.html"`  
`tables = pd.read_html(url)`  
`inspections = tables[0]`

### 3) Pull ratings via API
`load_dotenv()`  
`API_KEY = os.getenv("RESTAURANTS_API_KEY")`  
`params = {"api_key": API_KEY, "limit": 1000, "offset": 0}`  
(loop with offset=i*1000 to collect ~35k rows)

### 4) Cleaning
- Convert date strings to `datetime`  
- Normalize business names (uppercase, strip special chars)  
- Map `price_level` strings (`$$$$$`→5 … ` - No ratings yet`→0)  
- Extract ZIP codes, filter to NYC (10001–11697)  
- Drop duplicates and invalid rows  

### 5) Save to SQLite
`conn = sqlite3.connect("restaurant_data.db")`  
`inspections_clean.to_sql("inspections", conn, if_exists="replace", index=False)`  
`ratings_clean.to_sql("ratings", conn, if_exists="replace", index=False)`  
`conn.close()`

### 6) SQL analysis examples

**Average score by cuisine**  
SELECT CUISINE_DESCRIPTION, COUNT(*) AS total_inspections,
AVG(SCORE) AS avg_score
FROM inspections
GROUP BY CUISINE_DESCRIPTION
ORDER BY avg_score;

**Price level impact**  
SELECT DISTINCT inspections.DBA, inspections.SCORE, inspections.INSPECTION_DATE,
ratings.price_level, ratings.rating
FROM inspections
JOIN ratings
ON inspections.DBA = ratings.NAME
AND inspections.ZIPCODE = ratings.zip_code
WHERE ratings.price_level IS NOT NULL
ORDER BY inspections.SCORE DESC;

**Pass rate by inspection type**  
SELECT INSPECTION_TYPE,
COUNT(*) AS total_inspections,
AVG(CASE WHEN SCORE <= 13 THEN 1 ELSE 0 END) * 100 AS pass_rate,
AVG(SCORE) AS avg_score
FROM inspections
WHERE SCORE IS NOT NULL
GROUP BY INSPECTION_TYPE
ORDER BY pass_rate DESC;
