# Layoffs data cleaning and analysis

## [Dataset link](https://github.com/AlexTheAnalyst/MySQL-YouTube-Series/blob/main/layoffs.csv)
## Problem Statement
The dataset has 3 years of layoffs data. Clean the data and derive some insights from it.

## Data Cleaning
```sql
SELECT TOP 5 * FROM layoffs;
```
| company    | location       | industry     | total_laid_off | percentage_laid_off  | date       | stage     | country        | funds_raised_millions |
|------------|----------------|--------------|----------------|----------------------|------------|-----------|----------------|------------------------|
| Atlassian  | Sydney         | Other        | 500            | 0.0500000007450581   | 2023-03-06 | Post-IPO | Australia      | 210.00                 |
| SiriusXM   | New York City  | Media        | 475            | 0.0799999982118607   | 2023-03-06 | Post-IPO | United States  | 525.00                 |
| Alerzo     | Ibadan         | Retail       | 400            | NULL                 | 2023-03-06 | Series B | Nigeria        | 16.00                  |
| UpGrad     | Mumbai         | Education    | 120            | NULL                 | 2023-03-06 | Unknown  | India          | 631.00                 |
| Loft       | Sao Paulo      | Real Estate  | 340            | 0.150000005960464    | 2023-03-03 | Unknown  | Brazil         | 788.00                 |

1) Fix the decimal places in `percentage_laid_off`
```sql
ALTER TABLE layoffs
ALTER COLUMN percentage_laid_off DECIMAL(5,2);
```
| percentage_laid_off  |
|----------------------|
|0.05|
|0.08|
|NULL
|NULL|
|0.15|

---

2) Remove duplicates
```sql
WITH duplicate_rows AS (
  SELECT *, 
         ROW_NUMBER() OVER (PARTITION BY company, [location], industry, total_laid_off, percentage_laid_off, [date], stage, country, funds_raised_millions ORDER BY company) AS rn
  FROM layoffs
)
SELECT * 
FROM duplicate_rows
WHERE rn >1;
```
| company           | location     | industry      | total_laid_off | percentage_laid_off | date       | stage     | country        | funds_raised_millions | rn |
|------------------|--------------|---------------|----------------|----------------------|------------|-----------|----------------|------------------------|----|
| Casper           | New York City| Retail        | NULL           | NULL                 | 2021-09-14 | Post-IPO  | United States  | 339.00                 | 2  |
| Cazoo            | London       | Transportation| 750            | 0.15                 | 2022-06-07 | Post-IPO  | United Kingdom | 2000.00                | 2  |
| Hibob            | Tel Aviv     | HR            | 70             | 0.30                 | 2020-03-30 | Series A  | Israel         | 45.00                  | 2  |
| Wildlife Studios | Sao Paulo    | Consumer      | 300            | 0.20                 | 2022-11-28 | Unknown   | Brazil         | 260.00                 | 2  |
| Yahoo            | SF Bay Area  | Consumer      | 1600           | 0.20                 | 2023-02-09 | Acquired  | United States  | 6.00                   | 2  |

```sql
WITH duplicate_rows AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY company, [location], industry, total_laid_off, percentage_laid_off, [date], stage, country, funds_raised_millions ORDER BY company) AS rn
  FROM layoffs
)
DELETE FROM duplicate_rows
WHERE rn >1;
```
- 5 rows deleted
---

3) Populate NULL values in `industry`
```sql
SELECT t1.company, t1.industry, t2.company, t2.industry, ISNULL(t2.industry, t1.industry)
FROM layoffs t1
JOIN layoffs t2
  ON t1.company = t2.company AND t1.location = t2.location
WHERE t2.industry IS NULL AND
      t1.industry IS NOT NULL;
```
| company | industry      | company | industry | replacement_value |
|---------|---------------|---------|----------|-------------------|
| Carvana | Transportation | Carvana | NULL     | Transportation     |
| Carvana | Transportation | Carvana | NULL     | Transportation     |
| Airbnb  | Travel         | Airbnb  | NULL     | Travel             |
| Juul    | Consumer       | Juul    | NULL     | Consumer           |

```sql
UPDATE t2
SET industry = ISNULL(t2.industry, t1.industry)
FROM layoffs t1
JOIN layoffs t2
  ON t1.company = t2.company AND t1.location = t2.location
WHERE t2.industry IS NULL AND
      t1.industry IS NOT NULL;
```
---

4) Fix distinct values in `industry`
```sql
SELECT industry, COUNT(*) AS  total_rows
FROM layoffs
GROUP BY industry
ORDER BY 1 ASC;
```
| industry        | count |
|----------------|-------|
| NULL           | 1     |
| Aerospace      | 6     |
| Construction   | 16    |
| Consumer       | 117   |
| Crypto         | 99    |
| Crypto Currency| 2     |
| CryptoCurrency | 1     |

- Cryptocurrency industry has duplicated values. Need to fix it to be the same.
```sql
UPDATE layoffs
SET industry = 'Cryptocurrency'
WHERE industry LIKE 'Crypto%';
```
| industry        | count |
|----------------|-------|
| NULL           | 1     |
| Aerospace      | 6     |
| Construction   | 16    |
| Consumer       | 117   |
| Cryptocurrency | 102   |

---

5) Fix distinct values in `country`
```sql
SELECT country
FROM layoffs
ORDER BY country;
```
| country          |
|------------------|
| United Kingdom   |
| United States    |
| United States.   |
| Uruguay          |
| Vietnam          |

```sql
UPDATE layoffs
SET country = 'United States'
WHERE country = 'United States.';
```
| country          |
|------------------|
| United Kingdom   |
| United States    |
| Uruguay          |
| Vietnam          |

---
6) Delete unnecessary rows
```sql
DELETE FROM layoffs
WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;
```
- 361 rows deleted where `total_laid_off` and `percentage_laid_off` are both NULL. Without both columns, it's impossible to know if these companies even laid off anybody. Including these rows will alter the results of further analyses.
---
## EDA
1) Date range of data
```sql
SELECT MIN(date) AS min_date, MAX(date) AS max_date  FROM layoffs;
```
| min_date   | max_date   |
|------------|------------|
| 2020-03-11 | 2023-03-06 |

---

2) Number of cities by country where layoffs happened
```sql
WITH distinct_cities AS (
  SELECT DISTINCT country, [location] 
  FROM layoffs
)
SELECT country, COUNT([location]) AS numb_cities
FROM distinct_cities
GROUP BY country
ORDER BY numb_cities DESC;
```
| country       | numb_cities   |
|---------------|------------|
| United States | 86 |
| India         | 10 |
| Canada        | 9 |
| Germany       | 8 |
| Brazil        | 8|

- The United States has an unusually high number of cities compared to other countries. It’s worth investigating further to determine whether this is accurate or if there are other data issues that were missed during the initial cleaning process.
```sql
SELECT DISTINCT [location]
FROM layoffs
WHERE country = 'United States';
```
- Countries of some cities are wrongly classified as the United States. This needs to be corrected.
```sql
UPDATE layoffs
SET country = 'India'
WHERE [location] IN ('Chennai','New Delhi');
```
After further data cleaning:
| country       | numb_cities   |
|---------------|------------|
| United States | 67 |
| India         | 10 |
| Canada        | 9 |
| Germany       | 8 |
| Brazil        | 8|
- The US still has a significantly higher number of cities, but these are accurate results. This could be due to other reasons such as inaccurate reporting by other countries, the dataset is not exhaustive, or a bad economic situation in the US.
---
3) Total number of people laid off
```sql
SELECT FORMAT(SUM(total_laid_off),'N0') AS total_laid_off
FROM layoffs;
```
| total_laid_off | 
|----------------|
| 383,659        | 

---

4) Total number of people laid off by year
```sql
SELECT YEAR([date]) AS [year], FORMAT(SUM(total_laid_off),'N0') AS laid_off
FROM layoffs
WHERE [date] IS NOT NULL
GROUP BY YEAR([date])
ORDER BY 1;
```
| year | laid_off |
|------|----------|
| 2020 | 80,998   |
| 2021 | 15,823   |
| 2022 | 160,661  |
| 2023 | 125,677  |
- Even though we have less than 3 months of data in 2023, it has the second-highest layoff numbers, indicating a really bad year.
---

5) Total number of people laid off by country
```sql
SELECT country, FORMAT(SUM(total_laid_off),'N0') AS total_laid_off
FROM layoffs
GROUP BY country
ORDER BY SUM(total_laid_off) DESC;
```
| country        | total_laid_off |
|----------------|----------------|
| United States  | 256,107        |
| India          | 36,006         |
| Netherlands    | 17,220         |
| Sweden         | 11,264         |
| Brazil         | 10,491         |
| Germany        | 8,861          |

- These results for the United States, India, Brazil, and Germany are consistent with the number of locations in each country where layoffs happened. In the Netherlands and Sweden, the layoffs are high, but they happened in fewer locations. 
---
6) Total number of people laid off by location
```sql
SELECT country, [location], FORMAT(SUM(total_laid_off),'N0') AS laid_off
FROM layoffs
GROUP BY country, [location]
ORDER BY SUM(total_laid_off) DESC;
```
| country        | location         | laid_off |
|----------------|------------------|----------|
| United States  | SF Bay Area      | 125,601  |
| United States  | Seattle          | 34,743   |
| United States  | New York City    | 29,364   |
| India          | Bengaluru        | 21,787   |
| Netherlands    | Amsterdam        | 17,140   |

- The most layoffs happened in the SF Bay Area, followed by Seattle and NYC.
---
7) Top 5 companies that laid off the most employees
```sql
SELECT company, FORMAT(SUM(total_laid_off),'N0') AS laid_off
FROM layoffs
GROUP BY company
ORDER BY SUM(total_laid_off) DESC;
```
| company   | laid_off |
|-----------|----------|
| Amazon    | 18,150   |
| Google    | 12,000   |
| Meta      | 11,000   |
| Salesforce| 10,090   |
| Microsoft | 10,000   |

---

8) Total number of people laid off by industry
```sql
SELECT industry, FORMAT(SUM(total_laid_off),'N0') AS laid_off
FROM layoffs
GROUP BY industry
ORDER BY SUM(total_laid_off) DESC;
```
| industry        | laid_off |
|-----------------|----------|
| Consumer        | 45,182   |
| Retail          | 43,613   |
| Other           | 36,289   |
| Transportation  | 33,748   |
| Finance         | 28,344   |

---
9) Total number of people laid off by stage
```sql
SELECT stage, FORMAT(SUM(total_laid_off),'N0') AS laid_off
FROM layoffs
GROUP BY stage
ORDER BY SUM(total_laid_off) DESC;
```
| stage     | laid_off |
|-----------|----------|
| Post-IPO  | 204,132  |
| Unknown   | 40,716   |
| Acquired  | 27,576   |
| Series C  | 20,017   |
| Series D  | 19,225   |

---
10) Companies that laid off all of their employees
```sql
SELECT COUNT(*) AS complete_lay_off FROM layoffs
WHERE percentage_laid_off = 1;
```
| stage     | 
|-----------|
| 116  | 

---
11) Top 5 companies with the highest layoffs each year
```sql
WITH ranking_CTE AS (
SELECT company, YEAR([date]) AS year_laid_off, SUM(total_laid_off) AS laid_off, DENSE_RANK() OVER(PARTITION BY YEAR([date]) ORDER BY SUM(total_laid_off) DESC) AS rn
FROM layoffs
WHERE total_laid_off IS NOT NULL AND YEAR([date]) IS NOT NULL
GROUP BY company, YEAR([date])
)
SELECT *
FROM ranking_CTE
WHERE rn <= 5;
```
| company      | year_laid_off | laid_off | rn |
|--------------|---------------|----------|----|
| Uber         | 2020          | 7525     | 1  |
| Booking.com  | 2020          | 4375     | 2  |
| Groupon      | 2020          | 2800     | 3  |
| Swiggy       | 2020          | 2250     | 4  |
| Airbnb       | 2020          | 1900     | 5  |
| Bytedance    | 2021          | 3600     | 1  |
| Katerra      | 2021          | 2434     | 2  |
| Zillow       | 2021          | 2000     | 3  |
| Instacart    | 2021          | 1877     | 4  |
| WhiteHat Jr  | 2021          | 1800     | 5  |

---
12) Companies that laid off more than once
```sql
SELECT COUNT (*) AS companies_with_multiple_layoffs FROM (
SELECT 
    company, 
    COUNT(DISTINCT [date]) AS number_of_days_laid_off
FROM layoffs
GROUP BY company
HAVING COUNT(DISTINCT [date] ) > 1) AS sub_query;
```
| companies_with_multiple_layoffs      | 
|--------------|
| 282         |

---
