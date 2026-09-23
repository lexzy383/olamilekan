# Hi, I'm Mustapha Abdulsamod
### Data Analyst & Database Developer
Welcome to my portfolio! I specialize in data engineering, data cleaning, and optimizing relational database using SQL.

# Airbnb Data Cleaning & Transformation Pipeline (MySQL)

## 📌 Project Overview
Real-world datasets are rarely clean. In this project, I built a multi-stage data cleaning and transformation pipeline in **MySQL** using an unrefined Airbnb listings dataset. The goal was to take raw, unreliable text data and transform it into a high-integrity, production-ready database structured for fast business intelligence queries.

## 🛠️ Tech Stack & Concepts Used
* **Database Management System:** MySQL
* **Advanced SQL Techniques:** Common Table Expressions (CTEs), Window Functions (`ROW_NUMBER()`), Staging Tables
* **Data Quality Operations:** Deduplication, String Standardization, Type Casting, Data Validation

## 🏗️ The Data Cleaning Workflow

### 1. Staging & Safeguarding the Raw Data
**Why:** Modifying production or raw data directly is dangerous. I created physical staging tables (`listing_1` and `listings_2`) to isolate the destructive cleaning processes from the original source.
```sql
CREATE TABLE listing_1 LIKE listings;
INSERT INTO listing_1 SELECT * FROM listings;
```

### 2. Strategic Deduplication
**Why:** Standard CTE deletes are restricted in MySQL. I bypassed this limitation by building a second staging table (`listings_2`) with a physical `row_num` column, dynamically indexing duplicate rows using window functions partitioned by the unique listing identifiers.
```sql
INSERT INTO listings_2
SELECT *,
       ROW_NUMBER() OVER(
           PARTITION BY id, `name`, host_id 
           ORDER BY id
       ) AS row_num
FROM listing_1;

-- Safely purge duplicate rows
DELETE FROM listings_2 WHERE row_num > 1;
```

### 3. Text Standardization & Batch Performance
**Why:** Trailing and leading whitespaces corrupt data grouping (e.g., `"Manhattan "` vs `"Manhattan"`). Instead of executing multiple costly table scans, I combined individual updates into a single optimized `UPDATE` statement to save disk I/O.
```sql
UPDATE listings_2
SET `name` = TRIM(`name`),
    host_name = TRIM(host_name),
    neighbourhood = TRIM(neighbourhood);
```

### 4. Advanced Currency & Numeric Normalization
**Why:** The `price` column was stored as generic `TEXT` and mixed with currency symbols (`$`, `,`). Attempting to force-cast this into a numeric type would result in data truncation or corruption. 
* **The Solution:** I programmatically stripped the formatting symbols using nested `REPLACE` functions, eliminated blank values, and safely transformed the structural schema to an accurate financial data type (`DECIMAL(10,2)`).
```sql
-- Convert blank strings to NULL
UPDATE listings_2 SET price = NULL WHERE TRIM(price) = '';

-- Strip currency signs and commas
UPDATE listings_2
SET price = TRIM(REPLACE(REPLACE(price, '$', ''), ',', ''))
WHERE price IS NOT NULL;

-- Safe Type Transformation
ALTER TABLE listings_2 MODIFY COLUMN price DECIMAL(10,2);
```

### 5. Standardizing Text to MySQL DATE Type
**Why:** The `last_review` column was imported as `TEXT` (e.g., `'12-31-2025'`), preventing time-series analysis or date filters (like `DATEDIFF`).
* **The Solution:** I applied `STR_TO_DATE()` to match the layout of the incoming text strings, converted them into standard ISO date formats (`YYYY-MM-DD`), and updated the system architecture to a proper `DATE` type.
```sql
-- Convert text format into true date values
UPDATE listings_2
SET last_review = STR_TO_DATE(last_review, '%m-%d-%Y')
WHERE last_review IS NOT NULL AND last_review != '';

-- Alter the schema to enforce Date integrity
ALTER TABLE listings_2 MODIFY COLUMN last_review DATE;
```

### 6. Final Structural Cleanup
**Why:** Dropping operational logging columns (`row_num`) and entirely empty features (`neighbourhood_group`) to optimize storage footprints and finalize the clean dataset.
```sql
ALTER TABLE listings_2
DROP COLUMN neighbourhood_group,
DROP COLUMN row_num;
```

## 📈 Key Outcomes & Business Value
1. **Zero Data Corruption:** Safely converted unstructured currency text into precise decimal data types without data loss.
2. **Time-Series Ready:** Standardized date fields, opening up historical analytics tracking for stakeholders.
3. **Optimized I/O Execution:** Grouped row operations together, providing an framework that scales efficiently over larger production datasets.
