# Queries
## 1. Process and prepare database
### 1.1 Upload tables
- Project name: proyecto-03-riesgo
- Dataset name: dataset_banco
- Tables:
  - user_info
  - loans_outstanding
  - loans_detail
  - default

### 1.2 Identify and handle null values
Individual search
```sql
SELECT
  COUNT(*)
FROM
  `dataset_banco.user_info`
WHERE
  last_month_salary IS NULL
```
Search in all table columns
```sql
SELECT
  col_name,
  COUNT(1) nulls_count
FROM
  `dataset_banco.user_info` t,
  UNNEST(REGEXP_EXTRACT_ALL(TO_JSON_STRING(t), r'"(\w+)":null')) col_name
GROUP BY
  col_name
```

### 1.3 Identify and handle duplicated values
```sql
SELECT
  user_id,
  COUNT(user_id) AS cantidad
FROM
  `dataset_banco.user_info`
GROUP BY
  user_id
HAVING
  COUNT(*)>1
```

### 1.4 Identify variables with significant correlation
```sql
SELECT
  CORR(more_90_days_overdue, number_times_delayed_payment_loan_30_59_days) AS corr_90_30_59
FROM
  `dataset_banco.loans_detail`
```
### 1.5 Inconsistent values 
```sql
---Others->other and lower case all rows
SELECT
  * EXCEPT (loan_type),
  CASE
    WHEN LOWER(loan_type) = "others" THEN "other"
    ELSE LOWER(loan_type)
  END AS loan_type_lower
FROM
  `dataset_banco.loans_outstanding`
```

### 1.6 Identify outliers
```sql
WITH Stats AS (
  SELECT
    APPROX_QUANTILES(age, 4) AS quartiles
  FROM
    `dataset_banco.user_info`
)
SELECT
  age
FROM
  `dataset_banco.user_info`
CROSS JOIN
  Stats
WHERE
  age < quartiles[OFFSET(0)] - 1.5 * (quartiles[OFFSET(2)] - quartiles[OFFSET(0)])
  OR age > quartiles[OFFSET(2)] + 1.5 * (quartiles[OFFSET(2)] - quartiles[OFFSET(0)])
```

### 1.7 Create new variables
```sql
---From the loans_outstanding_cleaned table,create two columns to register each type of loan
SELECT
  user_id,
  COUNT(CASE
    WHEN loan_type='real estate' THEN 1
    ELSE NULL END) AS real_estate_loan,
  COUNT(CASE
    WHEN loan_type='other' THEN 1
    ELSE NULL END) AS other_loan,
  COUNT(loan_type) AS total_loan
FROM
  `dataset_banco.loans_outstanding_cleaned`
GROUP BY user_id
```
### 1.8 Join tables
Left join of user_info_nulls_flag, loans_types ,and loans_details
```sql
SELECT
  user_info_nulls_flag.*,
  loans_types.* EXCEPT(user_id),
  loans_detail.* EXCEPT(user_id)
FROM
  `dataset_banco.user_info_nulls_flag` AS user_info_nulls_flag
LEFT JOIN
  `dataset_banco.loans_types` AS loans_types
ON
  user_info_nulls_flag.user_id = loans_types.user_id
LEFT JOIN
  `dataset_banco.loans_detail` AS loans_detail
ON
  user_info_nulls_flag.user_id = loans_detail.user_id 
---425 null values. loans_types has only 35575 rows, 36000-35575 = 425
```
This table is saved as user_info_loans_details.

Replace null values with 0 
```sql
WITH
  remove_nulls AS (
  SELECT
    * REPLACE(IFNULL(real_estate_loan,0) AS real_estate_loan, IFNULL(other_loan,0) AS other_loan, IFNULL(total_loan,0) AS total_loan)
  FROM
    `dataset_banco.user_info_loans_details`)
SELECT
  *
FROM
  remove_nulls
```
This table is saved as full_table.

The table was then processed with the KNN_imputer (Google Colab). The result was saved as full_table_knn.

### 1.9 Final cleaning
```sql
SELECT user_id,
CASE WHEN age <=97 THEN age ELSE 97 END AS age,
CAST(CASE WHEN last_month_salary <= 15650 THEN last_month_salary ELSE 15650 END AS INT64) AS last_month_salary,
CAST(number_dependents AS INT64) AS number_dependents,
salary_null,
dependents_null,
default_flag,
real_estate_loan,
other_loan,
total_loan,
CASE WHEN more_90_days_overdue <=15 THEN more_90_days_overdue ELSE 15 END AS more_90_days_overdue,
ROUND(CAST(CASE WHEN using_lines_not_secured_personal_assets <= 1.31 THEN using_lines_not_secured_personal_assets ELSE 1.31 END AS FLOAT64),2) AS using_lines_not_secured_personal_assets,
ROUND(CAST(CASE WHEN debt_ratio <1 THEN debt_ratio ELSE 1 END AS FLOAT64),2) AS debt_ratio
FROM `dataset_banco.full_table_knn`
```
## 2. Quartiles
Calculate each quartile limits (upper and lower) per variable
```sql
SELECT
  APPROX_QUANTILES(age, 4)
FROM
  `dataset_banco.full_table_cleaned`
```
Then create quartiles or groups for each variable
```sql
WITH quartiles AS (
  SELECT user_id, 
  CASE 
    WHEN age <= 41 THEN 1
    WHEN age > 41 AND age <= 52 THEN 2
    WHEN age > 52 AND age <= 63 THEN 3
    WHEN age > 63 THEN 4
  END AS age_quartile,
  CASE 
    WHEN last_month_salary <= 3431 THEN 1
    WHEN last_month_salary > 3431 AND last_month_salary <= 5416 THEN 2
    WHEN last_month_salary > 5416 AND last_month_salary <= 8300 THEN 3
    WHEN last_month_salary > 8300 THEN 4
  END AS last_month_salary_quartile,
  CASE 
    WHEN number_dependents = 0 THEN 1
    WHEN number_dependents >= 1 THEN 2
  END AS number_dependents_quartile,
  CASE 
    WHEN real_estate_loan = 0 THEN 1
    WHEN real_estate_loan = 1 THEN 2
    WHEN real_estate_loan >1 THEN 3
  END AS real_estate_loan_quartile,
  CASE 
    WHEN other_loan <= 4 THEN 1
    WHEN other_loan > 4 AND other_loan <= 7 THEN 2
    WHEN other_loan > 7 AND other_loan <= 10 THEN 3
    WHEN other_loan > 10 THEN 4
  END AS other_loan_quartile,
  CASE 
    WHEN total_loan <= 5 THEN 1
    WHEN total_loan > 5 AND total_loan <= 8 THEN 2
    WHEN total_loan > 8 AND total_loan <= 11 THEN 3
    WHEN total_loan > 11 THEN 4
  END AS total_loan_quartile,
  CASE 
    WHEN more_90_days_overdue = 0 THEN 0
    WHEN more_90_days_overdue >= 1 THEN 1
  END AS more_90_days_overdue_quartile,
  CASE 
    WHEN using_lines_not_secured_personal_assets <= 0.03 THEN 1
    WHEN using_lines_not_secured_personal_assets > 0.03 AND using_lines_not_secured_personal_assets<= 0.15 THEN 2
    WHEN using_lines_not_secured_personal_assets > 0.15 AND using_lines_not_secured_personal_assets <= 0.55 THEN 3
    WHEN using_lines_not_secured_personal_assets > 0.55 THEN 4
  END AS using_lines_not_secured_personal_assets_quartile,
  CASE 
    WHEN debt_ratio <= 0.17 THEN 1
    WHEN debt_ratio > 0.17 AND debt_ratio<= 0.36 THEN 2
    WHEN debt_ratio > 0.36 AND debt_ratio <= 0.87 THEN 3
    WHEN debt_ratio > 0.87 THEN 4
  END AS debt_ratio_quartile,
  FROM `dataset_banco.full_table_cleaned`
)
SELECT a.*, q.* EXCEPT(user_id)
FROM `dataset_banco.full_table_cleaned` a
LEFT JOIN quartiles q ON a.user_id = q.user_id
```
This table is saved as full_table_quartile.

## 3. Relative risk
Calculate relative risk, transform into dummy variables, and calculate a total score. Also add columns to identify the corresponding range.

```sql
WITH 
  AgeRelativeRisk AS(
  SELECT age_quartile,
  ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS age_rr,
  CASE 
    WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0
  END AS age_rr_binary
  FROM `dataset_banco.full_table_quartiles`
  GROUP BY age_quartile
),

  SalaryRelativeRisk AS (
    SELECT 
      last_month_salary_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS salary_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS salary_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY last_month_salary_quartile
  ),

  DependentsRelativeRisk AS (
    SELECT 
      number_dependents_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS number_dependents_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS number_dependents_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY number_dependents_quartile
  ),

  TotalLoanRelativeRisk AS (
    SELECT 
      total_loan_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS total_loan_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS total_loan_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY total_loan_quartile
  ),

  OverdueRelativeRisk AS (
    SELECT 
      more_90_days_overdue_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS overdue_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS overdue_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY more_90_days_overdue_quartile
  ),

  LinesRelativeRisk AS (
    SELECT 
      using_lines_not_secured_personal_assets_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS lines_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS lines_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY using_lines_not_secured_personal_assets_quartile
  ),

  DebtRatioRelativeRisk AS (
    SELECT 
      debt_ratio_quartile,
      ROUND((COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))),2) AS debt_ratio_rr,
      CASE WHEN (COUNTIF(default_flag=1)/COUNT(*))/((683-COUNTIF(default_flag=1))/(36000-COUNT(*))) > 1.5 THEN 1 ELSE 0 END AS debt_ratio_rr_binary
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY debt_ratio_quartile
  ),


  AgeRanges AS(
    SELECT age_quartile,
    CONCAT(MIN(age),"-",MAX(age)) AS age_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY age_quartile
    ORDER BY age_quartile
  ),

  SalaryRanges AS(
    SELECT last_month_salary_quartile,
    CONCAT(MIN(last_month_salary),"-",MAX(last_month_salary)) AS salary_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY last_month_salary_quartile
    ORDER BY last_month_salary_quartile
  ),

  DependentsRanges AS(
    SELECT number_dependents_quartile,
    IF(number_dependents_quartile=1, "0", ">=1") AS dependents_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY number_dependents_quartile
    ORDER BY number_dependents_quartile
  ),

  TotalLoanRanges AS(
    SELECT total_loan_quartile,
    CONCAT(MIN(total_loan),"-",MAX(total_loan)) AS total_loan_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY total_loan_quartile
    ORDER BY total_loan_quartile
  ),

  OverdueRanges AS(
    SELECT more_90_days_overdue_quartile,
    IF(more_90_days_overdue_quartile=0, "0", ">=1") AS overdue_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY more_90_days_overdue_quartile
    ORDER BY more_90_days_overdue_quartile
  ),

  LinesRanges AS(
    SELECT using_lines_not_secured_personal_assets_quartile,
    CONCAT(ROUND(MIN(using_lines_not_secured_personal_assets),2), "-", ROUND(MAX(using_lines_not_secured_personal_assets),2)) AS lines_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY using_lines_not_secured_personal_assets_quartile
    ORDER BY using_lines_not_secured_personal_assets_quartile
  ),

  DebtRatioRanges AS(
    SELECT debt_ratio_quartile,
    CONCAT(ROUND(MIN(debt_ratio),2), "-", ROUND(MAX(debt_ratio),2)) AS debt_ratio_ranges,
    FROM `dataset_banco.full_table_quartiles`
    GROUP BY debt_ratio_quartile
    ORDER BY debt_ratio_quartile
  )


SELECT 
  a.*, 
  AgeRelativeRisk.age_rr AS age_rr, 
  AgeRelativeRisk.age_rr_binary AS age_rr_binary,
  SalaryRelativeRisk.salary_rr AS salary_rr,
  SalaryRelativeRisk.salary_rr_binary AS salary_rr_binary,
  DependentsRelativeRisk.number_dependents_rr AS number_dependents_rr,
  DependentsRelativeRisk.number_dependents_rr_binary AS number_dependents_rr_binary,
  TotalLoanRelativeRisk.total_loan_rr AS total_loan_rr,
  TotalLoanRelativeRisk.total_loan_rr_binary AS total_loan_rr_binary,
  OverdueRelativeRisk.overdue_rr AS overdue_rr,
  OverdueRelativeRisk.overdue_rr_binary AS overdue_rr_binary,
  LinesRelativeRisk.lines_rr AS lines_rr,
  LinesRelativeRisk.lines_rr_binary AS lines_rr_binary,
  DebtRatioRelativeRisk.debt_ratio_rr AS debt_ratio_rr,
  DebtRatioRelativeRisk.debt_ratio_rr_binary AS debt_ratio_rr_binary,
  age_rr_binary + salary_rr_binary + number_dependents_rr_binary + total_loan_rr_binary + overdue_rr_binary + lines_rr_binary + debt_ratio_rr_binary AS score,
  age_rr_binary + salary_rr_binary + number_dependents_rr_binary + total_loan_rr_binary + 3*overdue_rr_binary + 2*lines_rr_binary + debt_ratio_rr_binary AS score_weighted,

  AgeRanges.age_ranges AS age_ranges,
  SalaryRanges.salary_ranges AS salary_ranges,
  DependentsRanges.dependents_ranges AS dependents_ranges,
  TotalLoanRanges.total_loan_ranges AS total_loan_ranges,
  OverdueRanges.overdue_ranges AS overdue_ranges,
  LinesRanges.lines_ranges AS lines_ranges,
  DebtRatioRanges.debt_ratio_ranges AS debt_ratio_ranges,

FROM `dataset_banco.full_table_quartiles` a
LEFT JOIN AgeRelativeRisk ON a.age_quartile = AgeRelativeRisk.age_quartile
LEFT JOIN SalaryRelativeRisk ON a.last_month_salary_quartile = SalaryRelativeRisk.last_month_salary_quartile
LEFT JOIN DependentsRelativeRisk ON a.number_dependents_quartile = DependentsRelativeRisk.number_dependents_quartile
LEFT JOIN TotalLoanRelativeRisk ON a.total_loan_quartile = TotalLoanRelativeRisk.total_loan_quartile
LEFT JOIN OverdueRelativeRisk ON a.more_90_days_overdue_quartile = OverdueRelativeRisk.more_90_days_overdue_quartile
LEFT JOIN LinesRelativeRisk ON a.using_lines_not_secured_personal_assets_quartile = LinesRelativeRisk.using_lines_not_secured_personal_assets_quartile
LEFT JOIN DebtRatioRelativeRisk ON a.debt_ratio_quartile = DebtRatioRelativeRisk.debt_ratio_quartile

LEFT JOIN AgeRanges ON a.age_quartile = AgeRanges.age_quartile
LEFT JOIN SalaryRanges ON a.last_month_salary_quartile = SalaryRanges.last_month_salary_quartile
LEFT JOIN DependentsRanges ON a.number_dependents_quartile = DependentsRanges.number_dependents_quartile
LEFT JOIN TotalLoanRanges ON a.total_loan_quartile = TotalLoanRanges.total_loan_quartile
LEFT JOIN OverdueRanges ON a.more_90_days_overdue_quartile = OverdueRanges.more_90_days_overdue_quartile
LEFT JOIN LinesRanges ON a.using_lines_not_secured_personal_assets_quartile = LinesRanges.using_lines_not_secured_personal_assets_quartile
LEFT JOIN DebtRatioRanges ON a.debt_ratio_quartile = DebtRatioRanges.debt_ratio_quartile
```
This table is saved as full_table_score.

## 4. Final table for visualization with Looker Studio
```sql
SELECT
  user_id,
  age,
  last_month_salary,
  number_dependents,
  total_loan,
  more_90_days_overdue,
  ROUND(using_lines_not_secured_personal_assets,2) AS using_lines_not_secured_personal_assets,
  debt_ratio,
  age_rr,
  salary_rr,
  number_dependents_rr,
  total_loan_rr,
  overdue_rr,
  lines_rr,
  debt_ratio_rr,
  age_rr_binary,
  salary_rr_binary,
  number_dependents_rr_binary,
  total_loan_rr_binary,
  3*overdue_rr_binary AS overdue_rr_binary,
  2*lines_rr_binary AS lines_rr_binary,
  debt_ratio_rr_binary,
  age_ranges,
  salary_ranges,
  dependents_ranges,
  total_loan_ranges,
  overdue_ranges,
  lines_ranges,
  debt_ratio_ranges,
  CAST(default_flag AS INT64) AS default_flag,
IF
  (default_flag=1, "bad payer", "good payer") AS default_flag_category,
  score_weighted AS score
FROM
  `dataset_banco.full_table_score`
```
This table is saved as table_looker.
