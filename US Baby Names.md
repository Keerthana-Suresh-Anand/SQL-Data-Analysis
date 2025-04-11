# US Baby Names Analysis

## [Dataset link]([https://app.mavenanalytics.io/guided-projects/d7167b45-6317-49c9-b2bb-42e2a9e9c0bc](https://app.mavenanalytics.io/guided-projects/f71c0a2b-05f4-43fe-a80c-8f3f86964ccc))
## Problem Statement
The dataset has 30 years of baby names from 1980-2009. Track the changes in name popularity, compare popularity across decades and regions. 

## Understanding the tables
### Baby names table
Columns: State, Gender, Year, Name, Count

**1. Date range**
```sql
SELECT MIN([Year]) AS min_year, MAX([Year]) AS max_year FROM names_data;
```
|min_year|max_year|
|-----------|-----|
|1980|2009|

- The data covers the period from 1980 to 2009
---
**2. How many states does the data include?**
```sql
SELECT COUNT(DISTINCT [state]) AS total_states FROM names_data;
```
```sql
SELECT DISTINCT [state] FROM names_data
ORDER BY [state];
```
|total_states|
|-----------|
|51|

- The dataset includes baby names from 51 states. Upon closer inspection, it includes data from 50 states and Washington, D.C. which is a federal district but often gets included in lists for various administrative and data reporting purposses.
---
**3. How many unique names does the data have?**
```sql
SELECT FORMAT(COUNT(DISTINCT [Name]),'N0') AS numb_names FROM names_data;
```

|numb_names|
|-----------|
|22,240|

---
**4. How many names in each gender?**
```sql
SELECT Gender, FORMAT(COUNT(DISTINCT [Name]),'N0') AS total_names
FROM names_data
GROUP BY Gender;
```

|Gender|total_names|
|-----------|--|
|F|14,474|
|M|9,730|

-There are more unique female names than male names

---
**5. How many unisex names does the data have?**
```sql
SELECT COUNT(*) AS unisex_names 
FROM (
  SELECT DISTINCT [Name]
  FROM names_data
  GROUP BY [Name]
  HAVING COUNT(DISTINCT Gender) = 2
) AS sub_query;
```

|unisex_names|
|-----------|
|1964|
- The dataset has 1,964 names that are used for both genders
---
### Regions table
Columns: Region, state

**1. How many regions and states does the regions table have?**
```sql
SELECT COUNT(DISTINCT [state]) AS states FROM regions;
```
|states|
|-----------|
|51|

---

**2. How many regions does the table have?**
```sql
SELECT DISTINCT region
FROM regions;
```
New England and New_England are the same. The spacing needs to be cleaned up.
```sql
UPDATE regions
SET region = 'New_England'
WHERE region = 'New England';
```
| region        | 
|---------------|
| South         | 
| Midwest       | 
| Mountain      | 
| New_England   | 
| Pacific       | 
| Mid_Atlantic  | 

--- The table has 6 regions

**3. How many states are in each region?**
```sql
SELECT region, COUNT([state]) AS numb_states
FROM regions
GROUP BY region
ORDER BY numb_states DESC;
```
| region        | numb_states |
|---------------|-------------|
| South         | 16          |
| Midwest       | 12          |
| Mountain      | 8           |
| New England   | 6           |
| Pacific       | 5           |
| Mid_Atlantic  | 4           |

---

## Baby names analysis
**1. What are the most popular girl and boy names?**
```sql
SELECT TOP 2 Gender, [Name], FORMAT(SUM([Count]), 'N0') AS total_count
FROM names_data
WHERE Gender = 'M'
GROUP BY Gender, [Name]
ORDER BY SUM([Count]) DESC;
```
```sql
SELECT TOP 2 Gender, [Name], FORMAT(SUM([Count]), 'N0') AS total_count
FROM names_data
WHERE Gender = 'F'
GROUP BY Gender, [Name]
ORDER BY SUM([Count]) DESC;
```
|Gender|Name|total_count|
|-----------|-----|-----|
|M|Michael|1,376,418|
|M|Christopher|1,118,253|

|Gender|Name|total_count|
|-----------|-----|-----|
|M|Jessica|863,121|
|M|AShley|786,945|
- **Michael** is the most popular boys' name and **Jessica** is the most popular girls' name.
---

**2. How has the popularity of Michael and Jessica changed over time?**
```sql
WITH all_popularity AS (
  SELECT [Year], 
         [Name],
         SUM([Count]) AS total_count, 
         DENSE_RANK() OVER(PARTITION BY [Year] ORDER BY SUM([Count]) DESC) AS popularity
  FROM names_data
  WHERE Gender = 'M'
  GROUP BY  [Year], [Name]
)
SELECT [Year], [Name], popularity
FROM all_popularity
WHERE [Name] = 'Michael' 
ORDER BY [Name], [Year];
```
```sql
WITH all_popularity AS (
  SELECT [Year], 
         [Name],
         SUM([Count]) AS total_count, 
         DENSE_RANK() OVER(PARTITION BY [Year] ORDER BY SUM([Count]) DESC) AS popularity
  FROM names_data
  WHERE Gender = 'F'
  GROUP BY  [Year], [Name]
)
SELECT [Year], [Name], popularity
FROM all_popularity
WHERE [Name] = 'Jessica' 
ORDER BY [Name], [Year];
```
|Year|Name|popularity|
|-----|-----|-----|
|1980|Michael|1|
|1981|Michael|1|
|-|-|-|
|2008|Michael|2|
|2009|Michael|3|

|Year|Name|popularity|
|-----|-----|-----|
|1980|Jessica|3|
|1981|Jessica|2|
|-|-|-|
|2008|Jessica|60|
|2009|Jessica|77|

- Michael has remained one of the most popular boys' names over the years, while Jessica has slowly dropped in popularity.
---

**3. Which names have the biggest changes in popularity?**
```sql
WITH CTE_1980 AS (
  SELECT [Year], 
         [Name], 
         SUM([Count]) AS numb_babies, 
         DENSE_RANK() OVER (ORDER BY SUM([Count]) DESC) AS popularity_1980
  FROM names_data
  WHERE [Year] = 1980
  GROUP BY [Year], [Name]
),
CTE_2009 AS (
  SELECT [Year], 
         [Name], 
         SUM([Count]) AS numb_babies,
         DENSE_RANK() OVER (ORDER BY SUM([Count]) DESC) AS popularity_2009
  FROM names_data
  WHERE [Year] = 2009
  GROUP BY [Year], [Name]
)
SELECT TOP 5 t1.[Name], 
             t1.popularity_1980, 
             t2.popularity_2009, 
             t1.popularity_1980 - t2.popularity_2009 AS popularity_dif
FROM CTE_1980 t1
JOIN CTE_2009 t2
  ON t1.[Name] = t2.[Name] 
ORDER BY popularity_dif DESC;
```
```sql
WITH CTE_1980 AS (
  SELECT [Year], 
         [Name], 
         SUM([Count]) AS numb_babies, 
         DENSE_RANK() OVER (ORDER BY SUM([Count]) DESC) AS popularity_1980
  FROM names_data
  WHERE [Year] = 1980
  GROUP BY [Year], [Name]
),
CTE_2009 AS (
  SELECT [Year], 
         [Name], 
         SUM([Count]) AS numb_babies,
         DENSE_RANK() OVER (ORDER BY SUM([Count]) DESC) AS popularity_2009
  FROM names_data
  WHERE [Year] = 2009
  GROUP BY [Year], [Name]
)
SELECT TOP 5 t1.[Name], 
             t1.popularity_1980, 
             t2.popularity_2009, 
             t1.popularity_1980 - t2.popularity_2009 AS popularity_dif
FROM CTE_1980 t1
JOIN CTE_2009 t2
  ON t1.[Name] = t2.[Name] 
ORDER BY popularity_dif ASC;
```
| Name     | popularity_1980 |popularity_2009 | popularity_dif |
|----------|-----------|-----------|--------------------|
| Isabella | 1065      | 1         | 1064               |
| Ava      | 1026      | 18        | 1008               |
| Connor   | 1057      | 71        | 986                |
| Carter   | 1049      | 69        | 980                |
| Hayden   | 1068      | 89        | 979                |


| Name   | popularity_1980| popularity_2009 |popularity_dif |
|--------|-----------|-----------|--------------------|
| Misty  | 107       | 1277      | -1170              |
| Jill   | 136       | 1290      | -1154              |
| Dawn   | 137       | 1268      | -1131              |
| Kristy | 157       | 1270      | -1113              |
| Lori   | 154       | 1247      | -1093              |


- Isabella has the highest increase in popularity, whereas Misty had the highest drop.
---

**4. Most popular girl and boy names every year?**
```sql
WITH popularity_CTE AS (
  SELECT [Year], 
         [Name],
         Gender,
         SUM([Count]) AS numb_babies, 
         DENSE_RANK() OVER(PARTITION BY [Year], Gender ORDER BY SUM([Count]) DESC) AS popularity
  FROM names_data
  GROUP BY [Year], [Name], Gender
)
SELECT * 
FROM popularity_CTE
WHERE popularity <= 3;
```
| Year | Name         | Gender | numb_babies  | popularity |
|------|--------------|--------|--------|------|
| 1980 | Jennifer     | F      | 58,379 | 1    |
| 1980 | Amanda       | F      | 35,819 | 2    |
| 1980 | Jessica      | F      | 33,923 | 3    |
| 1980 | Michael      | M      | 68,680 | 1    |
| 1980 | Christopher  | M      | 49,091 | 2    |
| 1980 | Jason        | M      | 48,170 | 3    |
| 2009 | Isabella     | F      | 22,289 | 1    |
| 2009 | Emma         | F      | 17,889 | 2    |
| 2009 | Olivia       | F      | 17,429 | 3    |
| 2009 | Jacob        | M      | 21,162 | 1    |
| 2009 | Ethan        | M      | 19,839 | 2    |
| 2009 | Michael      | M      | 18,927 | 3    |

- Popular boys' names have remained relatively constant over the years, with Michael always in the top 3, while girls' names have been constantly changing.
---

**5. Most popular girl and boy names every decade?**
```sql
WITH CTE_decades AS (
  SELECT [Year], 
  		   [Name],
  		   Gender,
  		   [Count], 
  		   CASE
    			WHEN [Year] >= 1980 AND [Year] <= 1989 THEN '1980s'
    			WHEN [Year] >= 1990 AND [Year] <= 1999 THEN '1990s'
    			WHEN [Year] >= 2000 AND [Year] <= 2009 THEN '2000s'
    		 END AS decade
  FROM names_data
),
popularity_by_decade AS (
  SELECT decade,
         [Name], 
         Gender, 
         SUM([Count]) AS numb_babies,
         DENSE_RANK() OVER(PARTITION BY decade, Gender ORDER BY SUM([Count]) DESC) AS popularity
  FROM CTE_decades
  GROUP BY decade, [Name], Gender
)
SELECT * 
FROM popularity_by_decade
WHERE popularity <= 3;
```
| decade | Name        | Gender | numb_babies | popularity |
|--------|-------------|--------|-------------|-------------|
| 1980s  | Jessica     | F      | 469,452     | 1           |
| 1980s  | Jennifer    | F      | 440,845     | 2           |
| 1980s  | Amanda      | F      | 369,705     | 3           |
| 1980s  | Michael     | M      | 663,645     | 1           |
| 1980s  | Christopher | M      | 554,838     | 2           |
| 1980s  | Matthew     | M      | 458,918     | 3           |
| 1990s  | Jessica     | F      | 303,079     | 1           |
| 1990s  | Ashley      | F      | 301,798     | 2           |
| 1990s  | Emily       | F      | 237,222     | 3           |
| 1990s  | Michael     | M      | 462,302     | 1           |
| 1990s  | Christopher | M      | 360,196     | 2           |
| 1990s  | Matthew     | M      | 351,596     | 3           |
| 2000s  | Emily       | F      | 223,640     | 1           |
| 2000s  | Madison     | F      | 193,112     | 2           |
| 2000s  | Emma        | F      | 181,195     | 3           |
| 2000s  | Jacob       | M      | 273,746     | 1           |
| 2000s  | Michael     | M      | 250,471     | 2           |
| 2000s  | Joshua      | M      | 231,851     | 3           |

- Even though Michael is one of the popular boys' names, over the decades, the number of babies named Michael has decreased.
---
**6. How many babies were born in each region?**
```sql
SELECT r.region, FORMAT(SUM([Count]),'N0') AS numb_babies
FROM names_data n
LEFT JOIN regions r
  ON n.[State] = r.[state]
GROUP BY r.region
ORDER BY SUM([Count]);
```
| region        | numb_babies   |
|---------------|---------------|
| New England   | 4,269,213     |
| Mountain      | 6,282,217     |
| Mid Atlantic  | 13,742,667    |
| Pacific       | 17,540,716    |
| Midwest       | 22,676,130    |
| South         | 34,219,920    |

---

**7. What are the 3 most popular boy and girl names in each region?**
```sql
SELECT r.region, FORMAT(SUM([Count]),'N0') AS numb_babies
FROM names_data n
LEFT JOIN regions r
  ON n.[State] = r.[state]
GROUP BY r.region
ORDER BY SUM([Count]);
```
| region        | Gender | Name        | numb_babies | popularity |
|---------------|--------|-------------|-------------|------------|
| Mid_Atlantic  | F      | Jessica     | 122,614     | 1          |
| Mid_Atlantic  | F      | Ashley      | 99,433      | 2          |
| Mid_Atlantic  | F      | Jennifer    | 99,334      | 3          |
| Mid_Atlantic  | M      | Michael     | 263,070     | 1          |
| -  | -      | -     | -   | -          |
|-  | -     | - | -     | -          |
| South         | F      | Ashley      | 304,961     | 1          |
| South         | F      | Jessica     | 296,582     | 2          |
| South         | F      | Jennifer    | 211,753     | 3          |
| South         | M      | Christopher | 431,678     | 1          |
| South         | M      | Michael     | 429,351     | 2          |
| South         | M      | Joshua      | 372,409     | 3          |
- Jessica and Michael were the most popular girl and boy names in all the regions except the South, where they were in second place below Ashley and Christopher.
---
**8. What are the 10 most popular androgynous names?**
```sql
SELECT TOP 10 [Name], FORMAT(SUM([Count]),'N0') AS numb_babies
FROM names_data
GROUP BY [Name]
HAVING COUNT(DISTINCT Gender) = 2
ORDER BY SUM([Count]) DESC;
```
| Name        | Numb_Babies |
|-------------|-------------|
| Michael     | 1,382,856   |
| Christopher | 1,122,213   |
| Matthew     | 1,034,494   |
| Joshua      | 960,170     |
| Jessica     | 865,046     |
| Daniel      | 824,208     |
| David       | 819,479     |
| Ashley      | 792,865     |
| James       | 766,789     |
| Andrew      | 761,824     |

---
**9. Find the length of the shortest and longest names.**
```sql

SELECT MIN(LEN([Name])) AS shortest_name, MAX(LEN([Name])) AS longest_name FROM names_data;
```
| shortest_name        | longest_name |
|-------------|-------------|
| 2     | 15  |

---
**10. What are the most popular shortest and longest names?**
```sql
SELECT [Name], FORMAT(SUM([Count]),'N0') AS total_count
FROM names_data
WHERE LEN([Name]) = 2 OR LEN([Name]) = 15 
GROUP BY [Name]
ORDER BY LEN([Name]), SUM([Count]) DESC;
```
| Name        | total_count |
|-------------|-------------|
| Ty    | 29,205  |
| Bo    | 4,737  |
| Jo    | 1,713  |
| -   | -  |
| Franciscojavier    | 52  |
| Ryanchristopher    | 17 |
|Mariadelosangel|5|

- A lot more babies were given short names than the longest ones
---

**11. Find the state with the highest percent of the name Chris**
```sql
WITH chris_states AS(
  SELECT [State], [Name], SUM([Count]) AS chris_babies
  FROM names_data
  WHERE [Name] LIKE 'Chris'
  GROUP BY [State], [Name]
),
total_babies AS(
  SELECT [State] , SUM([Count]) AS all_babies
  FROM names_data
  GROUP BY [State]
)
SELECT TOP 3 t1.[State], ROUND(t1.chris_babies*100.00/t2.all_babies,4) AS chris_percent
FROM chris_states t1
JOIN total_babies t2
  ON t1.[State] = t2.[State]
ORDER BY chris_percent DESC;
```
| State        | chris_percent |
|-------------|-------------|
| LA    | 0.0335  |
| HI    | 0.0301  |
| NY    | 0.0296  |

- Louisiana had the highest percentage of babies named Chris (0.03%)
---
