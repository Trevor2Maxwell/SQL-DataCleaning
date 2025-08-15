# Data Cleaning
-----------
This is a collection of data cleaning ideas and how to use them.

### Eliminating Duplicate Records

Duplicate records are a prevalent issue that can distort analysis and inflate metrics
Identifying Duplicates: 

```  -- Identify duplicate rows based on customer_name and email
SELECT customer_name, email, COUNT(*)
FROM your_table_name -- Replace with your actual table name
GROUP BY customer_name, email
HAVING COUNT(*) > 1;
``` 

Strategies for Removing Duplicates:
Keeping the First Occurrence (or MIN/MAX ID): This approach involves identifying a unique identifier (like an id column) 
``` 
-- Remove duplicates while keeping the first occurrence (assuming 'id' is a unique primary key)
DELETE FROM your_table_name
WHERE id NOT IN (
    SELECT MIN(id)
    FROM your_table_name
    GROUP BY customer_name, email, order_date -- Columns that define a duplicate
);
``` 

Using ROW_NUMBER() with CTEs: This is a highly flexible 
``` 
-- Remove duplicates using ROW_NUMBER() and CTE, keeping the first occurrence
WITH CTE_Duplicates AS (
    SELECT
        *,
        ROW_NUMBER() OVER(PARTITION BY customer_name, email, order_date ORDER BY id) as rn
        -- PARTITION BY defines the columns that identify a duplicate set.
        -- ORDER BY id ensures a consistent row is kept if multiple IDs exist for the same logical record.
    FROM
        your_table_name
)
DELETE FROM CTE_Duplicates WHERE rn > 1;
``` 

### Effectively Handling Missing Values (NULLs)

Identifying NULLs: The IS NULL operator is the primary method to find records where a specific column lacks a value
``` 
-- Identify rows with missing values in a specific column
SELECT *
FROM table_name
WHERE column_name IS NULL;
``` 
Imputation Techniques:
``` 
-- Replace missing string values with 'N/A'
UPDATE table_name
SET column_name = 'N/A'
WHERE column_name IS NULL;

-- Replace missing numeric values with 0
UPDATE table_name
SET numeric_column = 0
WHERE numeric_column IS NULL;
``` 
Replacing with Aggregate Values (Mean, Median, Mode): 
``` 
-- Replace missing numeric values with the column's mean
UPDATE table_name
SET numeric_column = (SELECT AVG(numeric_column) FROM table_name WHERE numeric_column IS NOT NULL)
WHERE numeric_column IS NULL;
``` 
Using COALESCE or ISNULL (for SELECT statements): 
``` 
-- Display column_name, replacing NULLs with 'Unknown' for reporting
SELECT COALESCE(column_name, 'Unknown') AS cleaned_column
FROM table_name;

-- Delete rows where a critical column has missing values
DELETE FROM table_name
WHERE critical_column IS NULL;
``` 
#### The choice between deletion and imputation depends heavily on the percentage of missing data, 

**Standardizing Data Formats and Types**

**Inconsistent data formats** 
Standardization:
``` 
-- Standardize text to uppercase for consistent comparisons
UPDATE table_name
SET text_column = UPPER(text_column);

-- Remove leading/trailing whitespace from a text column
UPDATE table_name
SET text_column = TRIM(text_column);

-- Convert date string 'MM/DD/YYYY' to a consistent DATE format (SQL Server example)
UPDATE table_name
SET date_column = CONVERT(DATE, date_string_column, 101); -- 101 is mm/dd/yyyy format

-- Convert date string to a consistent DATE format (PostgreSQL/MySQL example)
UPDATE table_name
SET date_column = CAST(date_string_column AS DATE);
``` 

Ensuring Correct Data Types: 
``` 
-- Identify non-numeric entries in a column intended to be numeric (SQL Server example)
SELECT problematic_column
FROM table_name
WHERE ISNUMERIC(problematic_column) = 0;

-- Identify values that cannot be cast to a date (SQL Server TRY_CAST example)
SELECT string_date_column
FROM table_name
WHERE TRY_CAST(string_date_column AS DATE) IS NULL
AND string_date_column IS NOT NULL;

```



### Data Validation Checks
``` 

-- Find entries where 'age_column' is outside the valid range of 18 to 65
SELECT *
FROM table_name
WHERE age_column NOT BETWEEN 18 AND 65;
``` 
``` 
-- Find invalid entries not in a predefined list of 'status_column' values
SELECT *
FROM table_name
WHERE status_column NOT IN ('Active', 'Inactive', 'Pending', 'Completed');
``` 

### Uniqueness Enforcement:

``` 
-- Add a unique constraint to the 'email_address' column to prevent duplicates
ALTER TABLE table_name
ADD CONSTRAINT uc_email UNIQUE (email_address);
``` 

Format Validation:
``` 
-- Add a CHECK constraint to ensure 'product_code' starts with a letter followed by two digits
ALTER TABLE table_name
ADD CONSTRAINT ck_product_code CHECK (product_code LIKE '[A-Z][0-9][0-9]');

-- Add a CHECK constraint to ensure 'phone_number' length is between 10 and 15 characters
ALTER TABLE table_name
ADD CONSTRAINT ck_phone_length CHECK (LEN(phone_number) >= 10 AND LEN(phone_number) <= 15);
``` 
``` 
-- Extract first name and last name from a 'full_name' field (e.g., 'John Doe')
SELECT
    full_name,
    SUBSTRING(full_name, 1, CHARINDEX(' ', full_name) - 1) AS first_name,
    SUBSTRING(full_name, CHARINDEX(' ', full_name) + 1, LEN(full_name)) AS last_name
FROM employees;
``` 
``` 
-- Split a comma-separated list of tags into separate columns (PostgreSQL example)
SELECT
    article_id,
    SPLIT_PART(tags, ',', 1) AS tag1,
    SPLIT_PART(tags, ',', 2) AS tag2,
    SPLIT_PART(tags, ',', 3) AS tag3
FROM articles;

-- Using STRING_SPLIT to unpivot a delimited string into multiple rows (SQL Server example)
SELECT article_id, value AS tag
FROM articles
CROSS APPLY STRING_SPLIT(tags, ',');

``` 
``` 
-- Identify potential outliers using a simplified Z-score approach (values > 3 standard deviations from mean)
SELECT
    value_column,
    (value_column - AVG(value_column) OVER()) / STDEV(value_column) OVER() AS z_score
FROM
    your_data_table
WHERE
    ABS((value_column - AVG(value_column) OVER()) / STDEV(value_column) OVER()) > 3;
``` 


### Window Functions: 

Window functions fundamentally change the way analysts can interact with data. Unlike aggregate functions that return a single value per group (e.g., SUM with GROUP BY), 
window functions return a value for each row in the result set, based on a "window" of rows related to the current row.
``` 
-- Example: Calculate total sales per city (using PARTITION BY)
SELECT
    store_ID,
    day,
    city,
    gross_sales,
    SUM(gross_sales) OVER(PARTITION BY city) AS total_sales_in_city
FROM store_sales
WHERE day = 1;

-- Example: Running total of sales within each store, ordered by day
SELECT
    store_ID,
    day,
    gross_sales,
    SUM(gross_sales) OVER(PARTITION BY store_ID ORDER BY day) AS running_total_sales
FROM store_sales;

-- Cumulative sum of gross sales per store, by day
SELECT
    store_ID,
    day,
    gross_sales,
    SUM(gross_sales) OVER (PARTITION BY store_ID ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_sales
FROM store_sales;
``` 

Sliding Window Frame: 
``` 

-- 3-day moving average of gross sales per store
SELECT
    store_ID,
    day,
    gross_sales,
    AVG(gross_sales) OVER (PARTITION BY store_ID ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3_days
FROM store_sales;
``` 

### Ranking Functions

Ranking functions assign a rank to each row within its partition 
``` 
-- Assign a unique row number to employees within each department by salary (highest first)
SELECT
    employee_id,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_row_num
FROM employee_salaries;


-- Rank employees by salary within each department, with gaps for ties
SELECT
    employee_id,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employee_salaries;

-- Rank employees by salary within each department, without gaps for ties
SELECT
    employee_id,
    department,
    salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_dense_rank
FROM employee_salaries;


-- Divide employees into 4 salary quartiles within each department (1=highest, 4=lowest)
SELECT
    name,
    salary,
    NTILE(4) OVER (PARTITION BY dept ORDER BY salary DESC) AS salary_quartile
FROM employee;
``` 

### Distribution Functions for Relative Positioning

Distribution functions describe the relative standing or position of a row within a sorted partition

``` 
-- Calculate the percentile rank of each employee's salary within their department
SELECT
    employee_id,
    department,
    salary,
    PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary ASC) AS salary_percent_rank
FROM employee_salaries;

-- Calculate the cumulative distribution of sales for each store
SELECT
    store_ID,
    day,
    gross_sales,
    CUME_DIST() OVER (PARTITION BY store_ID ORDER BY gross_sales ASC) AS sales_cume_dist
FROM store_sales;

-- Running sum of sales per store (cumulative total)
SELECT
    store_ID,
    day,
    gross_sales,
    SUM(gross_sales) OVER (PARTITION BY store_ID ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM store_sales;

-- 7-day moving average of daily sales (sliding window)
SELECT
    sale_date,
    daily_sales,
    AVG(daily_sales) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS seven_day_moving_avg
FROM daily_sales_data;

``` 

### Combining Data Cleaning and Window Functions 

``` 
WITH CleanedSalesData AS (
    SELECT
        id,
        COALESCE(sales_amount, 0) AS cleaned_sales_amount, -- Handle missing values
        UPPER(product_category) AS cleaned_category,       -- Standardize text
        order_date
    FROM
        RawSalesData
    WHERE
        order_date IS NOT NULL -- Filter invalid dates
        -- Add more cleaning steps here (e.g., TRIM, duplicate removal via ROW_NUMBER())
),
AnalyzedSales AS (
    SELECT
        id,
        cleaned_sales_amount,
        cleaned_category,
        order_date,
        SUM(cleaned_sales_amount) OVER (PARTITION BY cleaned_category ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_category_sales,
        LAG(cleaned_sales_amount, 1, 0) OVER (PARTITION BY cleaned_category ORDER BY order_date) AS prev_category_sales
    FROM
        CleanedSalesData
)
SELECT * FROM AnalyzedSales;
``` 

