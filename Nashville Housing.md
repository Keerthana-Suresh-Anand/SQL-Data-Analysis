# Nashville Housing data cleaning

## [Dataset link](https://github.com/AlexTheAnalyst/PortfolioProjects/blob/main/Nashville%20Housing%20Data%20for%20Data%20Cleaning.xlsx)
## Problem Statement
The dataset has 56,000 rows of Nashville Housing data. The data has NULLs, missing values, incorrect datatypes, and addresses rolled into the same column. Clean the dataset so it can be used for further analysis.

## Nashville Housing data
```sql
SELECT TOP 5 * FROM NashvilleHousing;
```

| UniqueID | ParcelID          | LandUse       | PropertyAddress               | SaleDate   | SalePrice | LegalReference      | SoldAsVacant | OwnerName                        | OwnerAddress                         | Acreage | TaxDistrict              | LandValue | BuildingValue | TotalValue | YearBuilt | Bedrooms | FullBath | HalfBath |
|----------|-------------------|---------------|-------------------------------|------------|-----------|----------------------|---------------|------------------------------------|--------------------------------------|---------|---------------------------|-----------|---------------|-------------|-----------|----------|----------|----------|
| 53223    | 061 14 0 116.00   | SINGLE FAMILY | 3803 BURRUS  ST, NASHVILLE    | 2016-08-16 00:00:00 000| 333000    | 20160818-0086579     | No            | HERNDON, MARY C.                  | 3803  BURRUS ST, NASHVILLE, TN       | 0.25    | URBAN SERVICES DISTRICT  | 30000     | 157500        | 194000     | 1940      | 3        | 2        | 0        |
| 692      | 061 14 0 117.00   | SINGLE FAMILY | 3801  BURRUS ST, NASHVILLE    | 2013-02-21 00:00:00 000| 137000    | 20130301-0020568     | No            | DEVLIN, BRANDON                   | 3801  BURRUS ST, NASHVILLE, TN       | 0.18    | URBAN SERVICES DISTRICT  | 28200     | 93700         | 139400     | 1950      | 2        | 1        | 0        |
| 15609    | 061 14 0 120.00   | SINGLE FAMILY | 801  GILLOCK ST, NASHVILLE    | 2014-05-22 00:00:00 000 | 120000    | 20140523-0044761     | No            | ESPY, KATHERINE & WEBB, ERIC      | 801  GILLOCK ST, NASHVILLE, TN       | 0.27    | URBAN SERVICES DISTRICT  | 30000     | 123800        | 161400     | 1926      | 3        | 1        | 0        |
| 28234    | 061 14 0 120.00   | SINGLE FAMILY | 801  GILLOCK ST, NASHVILLE    | 2015-03-17 00:00:00 000| 216000    | 20150318-0023349     | No            | ESPY, KATHERINE & WEBB, ERIC      | 801  GILLOCK ST, NASHVILLE, TN       | 0.27    | URBAN SERVICES DISTRICT  | 30000     | 123800        | 161400     | 1926      | 3        | 1        | 0        |
| 8035     | 061 14 0 124.00   | SINGLE FAMILY | 3826  BURRUS ST, NASHVILLE    | 2013-09-26 00:00:00 000| 101100    | 20131003-0103831     | No            | LATTIMORE, SCOTT C. & EMILY O.    | 3826  BURRUS ST, NASHVILLE, TN       | 0.25    | URBAN SERVICES DISTRICT  | 30000     | 187300        | 217300     | 1948      | 3        | 2        | 0        |

## Data Cleaning
1) `SaleDate` column has no timestamp data. It is all 0s. Remove the timestamp.
```sql
ALTER TABLE NashvilleHousing
ALTER COLUMN SaleDate DATE;
```
| SaleDate   |
|------------|
| 2016-08-16 |
| 2013-02-21 |
| 2014-05-22 |
| 2015-03-17 |
| 2013-09-26 |

---
2) Fix NULLs in `PropertyAddress`
```sql
SELECT * FROM NashvilleHousing
WHERE PropertyAddress IS NULL;
```
- 35 rows have NULL values in the PropertyAddress column. From the other non-NULL rows, we observe that the same address has the same `ParcelID`. We'll try to populate the NULL values using other property addresses with the same `ParcelID`.
```sql
SELECT t1.ParcelID, t1.PropertyAddress, t2.ParcelID, t2.PropertyAddress
FROM NashvilleHousing t1
JOIN NashvilleHousing t2
  ON t1.ParcelID = t2.ParcelID AND t1.UniqueID != t2.UniqueID
WHERE t2.PropertyAddress IS NULL;
```
```sql
UPDATE t2
SET t2.PropertyAddress = ISNULL(t2.PropertyAddress, t1.PropertyAddress)
FROM NashvilleHousing t1
JOIN NashvilleHousing t2
  ON t1.ParcelID = t2.ParcelID AND t1.UniqueID != t2.UniqueID
WHERE t2.PropertyAddress IS NULL;
```
---
3) Split `PropertyAddress` and `OwnerAddress` into separate columns based on commas
```sql
SELECT PropertyAddress,
       SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1) AS property_street_name,
       SUBSTRING(PropertyAddress, CHARINDEX(',',PropertyAddress)+1, LEN(PropertyAddress)) AS property_city
FROM NashvilleHousing;
```
| PropertyAddress             | property_street_name| property_city |
|-----------------------------|---------------------|------------|
| 3803 BURRUS  ST, NASHVILLE  | 3803 BURRUS  ST     | NASHVILLE  |
| 3801  BURRUS ST, NASHVILLE  | 3801  BURRUS ST     | NASHVILLE  |
| 801  GILLOCK ST, NASHVILLE  | 801  GILLOCK ST     | NASHVILLE  |
| 801  GILLOCK ST, NASHVILLE  | 801  GILLOCK ST     | NASHVILLE  |

```sql
ALTER TABLE NashvilleHousing
ADD property_street_name VARCHAR(100);
```
```sql
UPDATE NashvilleHousing
SET property_street_name = SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1);
```
```sql
ALTER TABLE NashvilleHousing
ADD property_city VARCHAR(50);
```
```sql
UPDATE NashvilleHousing
SET property_city = SUBSTRING(PropertyAddress, CHARINDEX(',',PropertyAddress)+1, LEN(PropertyAddress));
```
```sql
SELECT OwnerAddress, 
       REPLACE(OwnerAddress, ',','.') AS modified_address, 
       PARSENAME(REPLACE(OwnerAddress, ',','.'),1) AS owner_state, 
       PARSENAME(REPLACE(OwnerAddress, ',','.'),2) owner_city, 
       PARSENAME(REPLACE(OwnerAddress, ',','.'),3) AS owner_street_name
FROM NashvilleHousing;
```
| OwnerAddress                        | modified_address                   | owner_state | Cowner_city      | owner_street_name         |
|------------------------------------|----------------------------------|-------|-----------|--------------------|
| 3803  BURRUS ST, NASHVILLE, TN     | 3803  BURRUS ST. NASHVILLE. TN   | TN    | NASHVILLE | 3803  BURRUS ST    |
| 3801  BURRUS ST, NASHVILLE, TN     | 3801  BURRUS ST. NASHVILLE. TN   | TN    | NASHVILLE | 3801  BURRUS ST    |
| 801  GILLOCK ST, NASHVILLE, TN     | 801  GILLOCK ST. NASHVILLE. TN   | TN    | NASHVILLE | 801  GILLOCK ST    |
| 801  GILLOCK ST, NASHVILLE, TN     | 801  GILLOCK ST. NASHVILLE. TN   | TN    | NASHVILLE | 801  GILLOCK ST    |
```sql
ALTER TABLE NashvilleHousing
ADD owner_streeet_name VARCHAR(100);
```
```sql
UPDATE NashvilleHousing
SET owner_streeet_name = PARSENAME(REPLACE(OwnerAddress, ',','.'),3);
```
```sql
ALTER TABLE NashvilleHousing
ADD owner_city VARCHAR(50);
```
```sql
UPDATE NashvilleHousing
SET owner_city = PARSENAME(REPLACE(OwnerAddress, ',','.'),2);
```
```sql
ALTER TABLE NashvilleHousing
ADD owner_state VARCHAR(10);
```
```sql
UPDATE NashvilleHousing
SET owner_state = PARSENAME(REPLACE(OwnerAddress, ',','.'),1);
```
---
4) `SoldAsVacant` has inconsistent values
```sql
SELECT SoldAsVacant, COUNT(*) AS total_values
FROM NashvilleHousing
GROUP BY SoldAsVacant;
```
| SoldAsVacant   | total_values  | 
|---------|--------------|
| N   | 399  | 
| Yes   | 4623   | 
| Y     | 52  |
| No    | 51403  |

```sql
UPDATE NashvilleHousing
SET SoldAsVacant = 'Yes'
WHERE SoldAsVacant = 'Y';
```
```sql
UPDATE NashvilleHousing
SET SoldAsVacant = 'No'
WHERE SoldAsVacant = 'N';
```
---
5) Delete duplicate rows
```sql
WITH CTE AS (
  SELECT *, 
         ROW_NUMBER() OVER(PARTITION BY ParcelID, PropertyAddress, SaleDate, SalePrice, LegalReference ORDER BY UniqueID) AS rn
  FROM NashvilleHousing
  )
DELETE FROM CTE
WHERE rn > 1;
```
- 104 rows deleted from the dataset
---
6) Delete unnecessary columns
```sql
ALTER TABLE NashvilleHousing
DROP COLUMN PropertyAddress, OwnerAddress;
```
- 2 columns deleted
---
