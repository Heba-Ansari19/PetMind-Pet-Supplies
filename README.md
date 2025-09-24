# PetMind-Pet-Supplies
PetMind Pet Supplies project analyzes one year of in-store sales to measure how repeat purchases impact revenue across animals and categories. I cleaned inconsistent entries, normalized sizes and prices, imputed missing sales with medians, and ran aggregations to reveal whether repeat buyers drive higher sales for dogs and cats. to improve repeat sales.
<br>
<br>
## PROJECT OVERVIEW
I analyzed PetMind’s one-year in-store sales stored in the `pet_supplies` table to determine how repeat purchases impact revenue and which animals and products to prioritize. PetMind sells both luxury items (like toys) and everyday items (like food). Management tested a repeat-sales strategy for a year and now wants a clear report showing whether repeat buyers generate more sales and which items for which animals I should focus on. I begin by inspecting and cleaning data so my comparisons are accurate, then compute aggregates that directly answer the core question: do repeat purchases increase sales for each animal, and which repeat-purchase products for cats and dogs deserve attention.

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
* PostgreSQL
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
I used this query to get a glance at the dataset. By inspecting the first 10 rows, I could understand the general structure of the table, the types of data each column contains, and spot any immediate inconsistencies or unusual values. This initial check helped me plan my data cleaning steps and gave me confidence about which columns might need more attention.

### 2. Exploring the category column

```sql
-- Checking category column for missing, NULL, or inconsistent values

SELECT DISTINCT category
FROM pet_supplies;
```
I ran this query to identify all unique values in the category column. This helped me detect invalid entries, like the '-' value, and see if there were missing or null values. Understanding this was important because the category is a key categorical feature for my analysis. I needed to clean this column by replacing invalid or missing values with 'Unknown', ensuring accurate aggregation later.

### 3. Exploring the animal column

```sql
-- Checking animal column for missing, NULL, or inconsistent values

SELECT DISTINCT animal
FROM pet_supplies;
```
I executed this query to review all animal types in the dataset. By confirming there were only Dog, Cat, Fish, and Bird and no missing values, I knew this column was reliable for analysis. This step ensured that I could directly use this column for grouping, aggregating sales, and comparing repeat purchases without needing cleaning.

### 4. Exploring the size column

```sql
-- Checking size column for missing, NULL, or inconsistent values

SELECT DISTINCT size
FROM pet_supplies;
```
I used this query to identify all unique values in the size column. The output showed capitalization errors and extra categories, which could lead to duplicate or inconsistent analysis. Recognizing these issues allowed me to normalize sizes to Small, Medium, Large, replacing invalid or inconsistent entries with 'Unknown'. This was crucial for accurate aggregation by size in later analysis.

### 5. Exploring the price column

```sql
-- Checking price column for 'unlisted' or inconsistent entries

SELECT price 
FROM pet_supplies
WHERE price ILIKE '%Un%';
```
I ran this query to find all products labeled as 'unlisted' in the price column. Price is a critical continuous variable for analyzing sales, calculating averages, and understanding revenue. Detecting these invalid entries helped me decide to replace them with 0, so I could perform reliable numerical analysis without errors or distortions.

### 6. Exploring the sales column

```sql
-- Checking for missing sales values

SELECT product_id, sales 
FROM pet_supplies
WHERE sales IS NULL;
```
I executed this query to verify if any sales records were missing. Sales is the key metric for my analysis, so identifying missing values was essential. The output showed no issues, giving me confidence that I could proceed with calculations like average, min, and max sales, knowing that the column was mostly clean.

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
I ran this aggregation to understand the range and distribution of sales in the dataset. Knowing the minimum and maximum sales helped me identify outliers, while the average and total sales provided a high-level view of performance. This step was important for later comparisons, especially when analyzing repeat purchases versus non-repeat purchases.

### 8. Exploring the rating column

```sql
-- Checking rating column for missing or inconsistent values

SELECT DISTINCT rating
FROM pet_supplies;
```
I executed this query to see all unique ratings given by customers. The output showed 10 distinct ratings but also some missing or null values. Detecting these gaps was important because rating is a discrete metric that reflects customer satisfaction, which would be used in correlation with repeat purchases and sales performance. I later replaced missing ratings with 0 to maintain consistency in analysis.

### 9. Exploring the repeat_purchase column

```sql
-- Checking repeat_purchase column for valid values

SELECT DISTINCT repeat_purchase
FROM pet_supplies;
```
I ran this query to confirm the values in the repeat_purchase column. The output showed 0 (No) and 1 (Yes), which matched expectations. This verification was critical because repeat purchases are the focus of my analysis, and knowing that this column was clean allowed me to directly use it for grouping and comparisons without further cleaning.

### 10. Understanding table structure

```sql
-- Reviewing table structure and data types of all columns

SELECT column_name, data_type 
FROM information_schema.columns 
WHERE table_name = 'pet_supplies';
```
I used this query to check the data types of all columns. Ensuring that each column has the correct type (e.g., numeric for sales and price, integer for product_id, nominal for categories) is essential for accurate analysis. This step also guided me on where I might need casting or conversion during data cleaning, so that calculations and aggregations could be performed without errors.

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
I cleaned the dataset by standardizing categorical fields like category and size, replacing missing or invalid entries with 'Unknown' to keep all products in the analysis. For numeric fields like price and sales, I converted values to consistent numeric types, rounded appropriately, and used the median to impute missing sales, avoiding distortion from extreme values. I also ensured all columns had the correct data types and included only relevant rows with valid repeat_purchase. These steps produced a consistent and reliable dataset, ready for meaningful analysis and insights.

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
I ran the query by grouping the data by animal and repeat_purchase to directly compare sales between repeat buyers and one-time buyers for each type of pet. This approach helped me answer the key question of whether products bought repeatedly generate higher sales, as it allowed me to examine the average sales within each group. Including the minimum and maximum sales provided additional context on the spread of sales, showing whether the average was influenced by a few top-selling products or if there were low-performing items that might need attention. Rounding the results to whole numbers made the output easier to interpret for stakeholders.

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
I applied a filter to focus on **cats and dogs** and only on products that were **bought repeatedly**, because management wanted to prioritize these for promotions, bundling, or stock planning next year. By selecting `product_id`, `sales`, and `rating`, I captured the essential information needed to **identify top revenue drivers** and **highly-rated repeat products**, which can directly inform merchandising and marketing strategies. Using the original table provided, the raw product list, giving a baseline view, while applying the same filter on the cleaned dataset ensures consistency and corrects any issues caused by unlisted prices or inconsistent categories. This step produced a **direct action list** of 552 products, making it easy to focus efforts on the most valuable items.

<br>
<br>

## INSIGHTS AND FINDINGS ~

* The raw data contained multiple quality issues that would bias results if left unaddressed: '-' values in category, many capitalization variants in size, 'unlisted' in price, and NULL rating values. Cleaning these was essential to produce reliable group comparisons.
* By producing a cleaned SELECT (Task 1), I retained 1,500 usable rows and avoided dropping records, which keeps revenue totals and product counts accurate.
* Imputing missing sales with the median preserved central tendency without letting extreme top sellers distort aggregates — this made average comparisons between repeat and non-repeat groups more robust.
* Replacing 'unlisted' price with 0 keeps rows but reduces average price metrics; this is an explicit business decision that affects interpretation. I flagged it for follow-up.
* Marking missing ratings as 0 flags unrated products; I treat these zeros as “unknown” during interpretation, so I do not misclassify products as low-rated when ratings are simply absent.
* The animal × repeat_purchase aggregation yields a compact, interpretable comparison (8 rows) that directly answers whether repeat buyers correspond to higher average sales by animal.
* The Cat/Dog repeat-product extract yields 552 products to use for targeted promotions, retention campaigns, or deeper profitability analysis.

<br>
<br>

## RECOMMENDATIONS

* Rank the 552 repeat products by sales and rating to create top-20 lists for marketing campaigns and stock prioritization.
* Replacing 'unlisted' with 0 preserves rows but can bias averages; either source the true prices from the ingestion system or impute prices based on the median within each category and size to reduce distortions.
* Treat 0 as a missing flag and exclude these from mean rating calculations, while reporting the count of unrated products so product teams can collect feedback.
* Create a documented, repeatable cleaning layer (SQL view or ETL step) to ensure that dashboards and reports consistently use the same logic.
* Conduct analyses such as (1) top-20 repeat products by sales and rating for cats and dogs, (2) animal × repeat_purchase aggregations on the cleaned view to compare with raw results, and (3) revenue-ranked cohorts or RFM analysis for repeat buyers to measure lifetime value potential.

<br>
<br>

## FINAL NOTES

This project demonstrated how SQL can transform raw, inconsistent pet product data into actionable business insights. By systematically cleaning and standardizing the dataset, PetMind can make informed decisions on promotions, inventory prioritization, and stock allocation. These insights enable the company to focus on repeat-purchase products, meet customer demand effectively, optimize revenue, and ensure profitability. Future extensions could include predictive sales modeling, repeat-customer lifetime value analysis, or seasonal demand forecasting to further refine marketing, inventory, and promotional strategies.


<br>
<br>





















