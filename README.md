# PetMind-Pet-Supplies
**PetMind Pet Supplies** project analyzes one year of in-store sales to measure **how repeat purchases impact revenue across animals and categories**. I cleaned inconsistent entries, normalized sizes and prices, imputed missing sales with medians, and ran aggregations to reveal whether repeat buyers drive higher sales for dogs and cats. to improve repeat sales.
<br>
<br>
## PROJECT OVERVIEW
I analyzed PetMind’s one-year in-store sales stored in the `pet_supplies` table to determine **how repeat purchases impact revenue and which animals and products to prioritize**. PetMind sells both **luxury items** (like toys) and **everyday items** (like food). Management tested a repeat-sales strategy for a year and now wants a clear report showing **whether repeat buyers generate more sales and which items for which animals I should focus on**. I begin by inspecting and cleaning data so my comparisons are accurate, then compute aggregates that directly answer the core question: do repeat purchases increase sales for each animal, and which repeat-purchase products for cats and dogs deserve attention.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/11ffaf46-fe67-443b-b020-47d5e29be30c" />

<br>
<br>

## OBJECTIVE / KEY QUESTIONS
* Do repeat purchases increase total sales for each animal type (Dog, Cat, Fish, Bird)?

<br>

## DATASET
* Source: internal PetMind retail database
* Table: `pet_supplies`
  
  | Column Name      | Data Type  | Criteria                                            |
  | ---------------- | ---------- | --------------------------------------------------- |
  | product\_id      | Nominal    | Unique identifier of the product                    |
  | category         | Nominal    | Housing, Food, Toys, Equipment, Medicine, Accessory |
  | animal           | Nominal    | Dog, Cat, Fish, Bird                                |
  | size             | Ordinal    | Small, Medium, Large                                |
  | price            | Continuous | Positive numeric, rounded to 2 decimals             |
  | sales            | Continuous | Total sales last year, rounded to 2 decimals        |
  | rating           | Discrete   | 1–10                                                |
  | repeat\_purchase | Nominal    | 0 (No), 1 (Yes)                                     |
  
<br>

## TOOLS AND SKILLS USED 
* [PostgreSQL](https://www.postgresql.org/download/)
* Concepts used: CTEs, `PERCENTILE_CONT (median)`, `COALESCE`, `NULLIF`, `CASE`, `UPPER/LOWER/INITCAP`, casting, aggregation (`AVG`, `MIN`, `MAX`, `SUM`), `GROUP BY`, `ORDER BY`, `ROUND`, `LIMIT`.
* Techniques used: data cleaning & normalization, missing-value imputation (median for sales, 0 for price as specified, 0 for rating), data type casting, aggregation & segmentation, exploratory distinct checks, filtering invalid rows, building analysis-ready SELECTs (no UPDATE to original table).
  
<br>

## ANALYSIS & APPROACH

### 1. Overview of the table

```sql
-- Overview of the pet_supplies table: preview first 10 rows

SELECT * 
FROM pet_supplies 
LIMIT 10;
```
I used this query to get a glance at the dataset. By inspecting the first 10 rows, I could understand the **general structure of the table, the types of data each column contains, and spot any immediate inconsistencies or unusual values**. This initial check helped me plan my data cleaning steps and gave me confidence about which columns might need more attention.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/023eb6e0-1d5a-4f40-9934-d7bfe93d0373" />

### 2. Exploring the category column

```sql
-- Checking category column for missing, NULL, or inconsistent values

SELECT DISTINCT category
FROM pet_supplies;
```
I ran this query to **identify all unique values in the `category` column**. This helped me detect invalid entries, like the '-' value, and see if there were missing or null values. Understanding this was important because the category is a key categorical feature for my analysis. I needed to clean this column by **replacing invalid or missing values with 'Unknown'**, ensuring accurate aggregation later.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/d8929577-d1ca-4740-ac92-be4972fc16f2" />

### 3. Exploring the animal column

```sql
-- Checking animal column for missing, NULL, or inconsistent values

SELECT DISTINCT animal
FROM pet_supplies;
```
I executed this query to review all `animal` types in the dataset. By confirming there were **only Dog, Cat, Fish, and Bird and no missing values**, I knew this column was reliable for analysis. This step ensured that I could directly use this column for grouping, aggregating sales, and comparing repeat purchases without needing cleaning.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/4cea428e-723d-44bd-976c-c3375fd9c59c" />

### 4. Exploring the size column

```sql
-- Checking size column for missing, NULL, or inconsistent values

SELECT DISTINCT size
FROM pet_supplies;
```
I used this query to identify all unique values in the `size` column. The output showed **capitalization errors and extra categories**, which could lead to duplicate or inconsistent analysis. Recognizing these issues allowed me to **normalize sizes to Small, Medium, Large, replacing invalid or inconsistent entries with 'Unknown'**. This was crucial for accurate aggregation by size in later analysis.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/0dd6d614-b5d0-42d9-ab80-e4f9ccdbee91" />

### 5. Exploring the price column

```sql
-- Checking price column for 'unlisted' or inconsistent entries

SELECT price 
FROM pet_supplies
WHERE price ILIKE '%Un%';
```
I ran this query to find all `products` labeled as **'unlisted'** in the price column. Price is a critical continuous variable for analyzing sales, calculating averages, and understanding revenue. Detecting these invalid entries helped me decide to **replace them with 0**, so I could perform reliable numerical analysis without errors or distortions.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/85268ab4-c51e-4ca2-b0f7-1d3c4a7a7ea1" />

### 6. Exploring the sales column

```sql
-- Checking for missing sales values

SELECT product_id, sales 
FROM pet_supplies
WHERE sales IS NULL;
```
I executed this query to verify if any `sales` records were missing. Sales is the **key metric for my analysis**, so identifying missing values was essential. The output showed **no issues**, giving me confidence that I could proceed with calculations like average, min, and max sales, knowing that the column was mostly clean.

### 7. Aggregating sales values

```sql
-- Getting minimum, maximum, average, and total sales

SELECT 
    MIN(sales),
    MAX(sales),
    AVG(sales),
    SUM(sales)
FROM pet_supplies;
```
I ran this aggregation to **understand the range and distribution of `sales` in the dataset**. Knowing the minimum and maximum sales helped me **identify outliers**, while the average and total sales provided a **high-level view of performance**. This step was important for later comparisons, especially when analyzing repeat purchases versus non-repeat purchases.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/b2dc2c11-a3a7-4ed5-a166-9199398d3031" />

### 8. Exploring the rating column

```sql
-- Checking rating column for missing or inconsistent values

SELECT DISTINCT rating
FROM pet_supplies;
```
I executed this query to see all unique `ratings` given by customers. The output showed **10 distinct ratings, but also some missing or null values**. Detecting these gaps was important because rating is a discrete metric that reflects customer satisfaction, which would be used in correlation with repeat purchases and sales performance. I later **replaced missing ratings with 0 to maintain consistency in the analysis**.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/2d9c27c0-09b6-464e-ab15-4638416af89a" />

### 9. Exploring the repeat_purchase column

```sql
-- Checking repeat_purchase column for valid values

SELECT DISTINCT repeat_purchase
FROM pet_supplies;
```
I ran this query to confirm the values in the `repeat_purchase` column. The output showed **0 (No) and 1 (Yes)**, which matched expectations. This verification was critical because repeat purchases are the focus of my analysis, and knowing that this column was clean allowed me to directly use it for grouping and comparisons without further cleaning.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/986cb449-7436-4197-b431-c23a76707c9d" />

### 10. Understanding table structure

```sql
-- Reviewing table structure and data types of all columns

SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'pet_supplies';
```
I used this query to check the **data types of all columns**. Ensuring that each column has the correct type (e.g., numeric for sales and price, integer for product_id, nominal for categories) is essential for accurate analysis. This step also guided me on where I might need **casting or conversion during data cleaning, so that calculations and aggregations could be performed without errors.**

<img width="800" alt="image" src="https://github.com/user-attachments/assets/024a0b1a-3540-4e37-9e9d-f80d819a60f7" />


After thoroughly exploring each column of the pet_supplies table, we observed the following:

* `product_id`: This column was **perfect with no missing values or data type errors**, providing a strong, unique identifier for each product. No cleaning was needed here.
* `category`: **Several entries were missing or represented as '-'**, which could mislead aggregations. We replaced these with 'Unknown', ensuring all products remain visible and accurately grouped.
* `animal`: **All values were valid (Dog, Cat, Fish, Bird) with no missing entries**, allowing direct grouping and analysis without cleaning.
* `size`: The column had **capitalization inconsistencies and unexpected values**, causing fragmentation in analysis. We standardized values to Small, Medium, Large, and mapped invalid entries to 'Unknown'.
* `price`: Some products were labeled **'unlisted'**, which cannot be used in numeric analysis. We replaced these with 0, cast to numeric, and rounded to 2 decimals, keeping all rows while flagging potential bias in average price.
* `sales`: Mostly complete, but **any missing values were planned to be replaced with the median**, preserving central tendency without being affected by extreme values.
* `rating`: Some products had **missing ratings**, which could skew quality analysis. We replaced NULLs with 0 to flag unrated products while retaining all rows.
* `repeat_purchase`: Values were **consistent as 0 or 1, with no missing entries**, enabling clear segmentation of repeat versus one-time buyers.

By addressing these issues—replacing missing values, standardizing text, and correcting numeric formats—we created a **clean and reliable dataset**. This ensures accurate analysis of `sales`, `repeat_purchases`, product popularity, and animal-specific trends, providing a solid foundation for business insights.


### 11. Clean the pet_supplies

```sql
--- CTE sales_median

WITH sales_median AS (
	SELECT 
		PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY sales) :: NUMERIC AS median_sales
	FROM public.pet_supplies
	WHERE sales IS NOT NULL
)

---------------------------------------------------------------------------------------------------------------------------

SELECT 
	product_id::INTEGER,

  	-- Replace '-' and NULL in category with 'Unknown'
  	COALESCE(NULLIF(category, '-'), 'Unknown') AS category,

	--- No Missing values
	animal,

	-- Normalize size capitalization and replace invalid or NULL with 'Unknown'
	CASE
  		WHEN UPPER(size) IN ('SMALL', 'MEDIUM', 'LARGE') THEN INITCAP(LOWER(size))
  		ELSE 'Unknown'
	END AS size,

	-- Replace 'unlisted' or NULL price with 0, cast to numeric, round to 2 decimals
	ROUND(COALESCE(NULLIF(price, 'unlisted') :: NUMERIC, 0), 2) AS price,

	-- Replace NULL sales with median sales from CTE, round 2 decimals 
	ROUND(COALESCE(sales, (SELECT median_sales FROM sales_median)) :: NUMERIC, 2) AS sales,

	--- Replace NULL values with 0
	COALESCE(rating, 0)::INTEGER AS rating,

	-- Filtered in WHERE clause, so no COALESCE here 
	repeat_purchase
	
FROM pet_supplies
WHERE repeat_purchase IN (0, 1);
```
I cleaned the dataset by **standardizing categorical fields like category and size, replacing missing or invalid entries with 'Unknown'** to keep all products in the analysis. For numeric fields like **price and sales**, I converted values to **consistent numeric types, rounded appropriately, and used the median to impute missing sales**, avoiding distortion from extreme values. I also ensured all columns had the **correct data types** and included only relevant rows with valid repeat_purchase. These steps produced a consistent and reliable dataset, ready for meaningful analysis and insights.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/ed88cbf4-94d8-4e90-a870-eb541a08d58d" />

### 12. Compare sales by animal and repeat_purchase (avg, min, max)

```sql
-- animal_sales

SELECT
	animal,
	repeat_purchase,
	ROUND(AVG(sales) :: NUMERIC) AS avg_sales,
	ROUND(MIN(sales) :: NUMERIC) AS min_sales,
	ROUND(MAX(sales) :: NUMERIC) AS max_sales
	
FROM pet_supplies
GROUP BY animal, repeat_purchase
ORDER BY animal, repeat_purchase;
```
I ran the query by grouping the data by `animal` and `repeat_purchase` to directly **compare sales between repeat buyers and one-time buyers for each type of pet**. This approach helped me answer the key question of **whether products bought repeatedly generate higher sales**, as it allowed me to examine the average sales within each group. Including the **minimum and maximum sales provided additional context on the spread of sales**, showing whether the **average was influenced by a few top-selling products or if there were low-performing items that might need attention**. Rounding the results to whole numbers made the output easier to interpret for stakeholders.

<img width="800" alt="image" src="https://github.com/user-attachments/assets/e8e4e0cb-00ef-4362-b648-d0d2855d20ec" />

### 13. Extract repeat-purchased products for cats and dogs

```sql
-- popular_pet_products

SELECT
	product_id,
	sales,
	rating
	
FROM pet_supplies
WHERE (animal = 'Cat' OR animal = 'Dog')
	AND repeat_purchase = 1;
```
I applied a filter to focus on **cats and dogs** and only on products that were **bought repeatedly**, because management wanted to prioritize these for **promotions, bundling, or stock planning next year**. By selecting `product_id`, `sales`, and `rating`, I captured the essential information needed to **identify top revenue drivers** and **highly-rated repeat products**, which can directly inform merchandising and marketing strategies. 

<img width="800" alt="image" src="https://github.com/user-attachments/assets/0903cd23-2e4a-416a-b06a-f4a4da20b9a8" />

<br>
<br>

## INSIGHTS AND FINDINGS

* The raw dataset had **multiple quality issues**, including '-' in category, inconsistent size capitalization, 'unlisted' prices, and missing ratings, which **could bias results** if left unaddressed.
* After cleaning, **1,500 rows were retained** with standardized categories, sizes, median-imputed sales, 0 for unlisted prices, and 0 for missing ratings, **ensuring reliable aggregates and accurate revenue totals.**
* Grouping by animal and repeat_purchase and calculating average, minimum, and maximum sales provided a **clear view of repeat buyers’ impact**, producing 8 interpretable rows that highlighted **higher sales for repeat products.**
* Filtering for **cats and dogs with repeat purchases produced 552 products with sales and ratings**, offering a direct action list for **promotions, stock prioritization, and profitability analysis.**
* The **cleaning and standardization** process created a repeatable framework for future analysis, ensuring **robust, actionable insights for revenue, marketing, and inventory planning.**

<br>

## RECOMMENDATIONS

* Rank the **552 repeat products for cats and dogs by sales and rating** to create top-20 lists. These lists can **guide marketing campaigns and stock allocation to maximize revenue**.
* Replacing 'unlisted' with 0 preserves rows but may bias average price metrics. Consider **sourcing true prices from the ingestion system or imputing based on median values by category and size to reduce distortions**.
* Treat 0 as a missing flag and exclude these from mean rating calculations. Reporting the **number of unrated products helps product teams collect feedback and improve future product evaluations**.
* Create a documented SQL view or ETL step to standardize data cleaning. This ensures that dashboards, reports, and future analyses consistently use the same logic.
* Perform targeted follow-up analyses:
	* Identify the **top-20 repeat products by sales and rating for cats and dogs** to focus promotions and stock priorities.
	* Compare **animal × repeat_purchase aggregations** on the cleaned view against the raw results to validate trends.
	* Conduct **revenue-ranked cohorts or RFM analysis** for repeat buyers to measure lifetime value and inform customer retention strategies.

<br>

## FINAL NOTES

This project demonstrated how SQL can transform raw, inconsistent pet product data into actionable business insights. By systematically cleaning and standardizing the dataset, PetMind can make **informed decisions on promotions, inventory prioritization, and stock allocation.** These insights enable the company to focus on repeat-purchase products, meet customer demand effectively, optimize revenue, and ensure profitability. Future extensions could include predictive sales modeling, repeat-customer lifetime value analysis, or seasonal demand forecasting to further refine marketing, inventory, and promotional strategies.

<br>

## REFERENCE

* [DataCamp](https://app.datacamp.com/)
<br>
<br>
