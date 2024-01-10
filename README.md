# Credit Score Based on the Risk of Non-Payment

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/2a6e3bc1-977a-4d6e-8c51-c2b98f2abb7a)

---

## Tabla of Contents
- [Introduction](#introduction)
- [Tools](#tools)
- [Data Cleaning](#data-cleaning)
- [Models](#models)
- [Results](#results)
- [Visualization](#visualization)
- [Conclusions and Recommendations](#conclusions-and-recommendations)

## Introduction
The "Super Caja" bank is facing an increasing amount of credit requests. In this context, it is necessary to improve the credit request revision system, in order to save time-resources and reduce the risk of non-payment. To accomplish these tasks, the main goal of this project was to calculate the relative risk associated to several user variables and estimate a global score to accurately differenciate the bad payers from the good ones.

## Tools
- SQL (Google BigQuery)
- Python ([Google Colab](link))
- Looker Studio

## Data Cleaning
### Drop Columns
First of all, to guarantee non-biased results, the `sex` variable was discarded. 

Next, correlation between variables was checked. This led to discarding the `number_times_delayed_payment_loan_30_59_days` and the `number_times_delayed_payment_loan_60_89_days` variables, which had a high positive correlation with `more_90_days_overdue`.

Finally, the `loan_type` variable was split into three columns: 
- `real_estate_loan`: counts the Real Estate loans per user.
- `other_loan`: counts the other type loans per user.
- `total_loan`: counts the total loans per user.

As these variables are correlated, later in the project it was determined if it was better to use the first two columns or only the `total_loan` column.

### Null values
The original tables contained 7199 null values in the `last_month_salary` column and 943 in the `number_dependents` column. Once it was determined that the null values were randomly located, the K-Nearest Neighbors (KNN) algorithm was used to predict and impute those missing values.

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/94786f27-d05d-4acf-bb51-2e619a0064d6)

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/e7c00b68-9911-4155-9f0d-89e65cf08e33)

### Outliers
The outliers were handled capping the outliers to the maximum value using the IQR range:
- `age`: any value higher than 97 was set to 97.
- `last_month_salary`: any value higher than 15650 was set to 15650.
- `more_90_days_overdue`: any value higher than 1 was set to 1.
- `using_lines_not_secured_personal_assets`: any value higher than 1.31 was set to 1.31.
- `debt_ratio`: any value higher than 1 was set to 1.

Using this approach, the original 36000 rows were preserved.

## Models

### Relative Risk Model

#### Quartiles
To prepare the data for further analysis and find patterns per group, each variable was divided into quartiles. If the variable already contained a considerable amount of zeros, the creation of only two or three categories was preferred over the creation of quartiles. Then, the relative risk was calculated in each group.

#### Relative Risk
The relative risk was calculated as follows:

$Relative\ risk = {Incidence\ risk\ among\ an\ exposed\ group \over Incidence\ risk\ among\ a\ non-exposed\ group}$

||Disease (case)|No disease (control)|Total|
|--------|--------|--------|--------|
|Exposed|a|b|a+b|
|Non-exposed|c|d|c+d|

$Relative\ risk = {a/a+b \over c/c+d}$

In the context of this project, the exposure factors were the user variables (age, salary, overdue, etc.) and the event or disease was the `default_flag` column, meaning bad antecedents with credit. So, for example, in the case of the `age` variable, the calculation for the relative risk in the first quartile was stated as:

||default_flag=1|default_flag=0|Total|
|--------|--------|--------|--------|
|Q1(21-41 years)|311|8764|9075|
|Q2, Q3, Q4 (other age ranges)|372|26553|26925|

$Relative\ risk = {311/9075 \over 372/26925} = {0.034 \over 0.013} = 2.48$  

This number means that the youngest clients are **2.48** times more likely to be bad payers than older clients. 

The corresponding calculus was estimated for each quartile or category and for each variable considered in the model. The `total_loan` column was preserved since it contained the highest relative risk value compared to the `real_estate_loan` and `other_loan` columns.

#### Credit Score
The relative risk values were transformed to binary values (dummy variables) using the following criteria:
- 1 if the relative risk was higher than 1.4
- 0 if the relative risk was equal to or lower than 1.4
None of the `debt_ratio` quartiles had a relative risk value higher than 1.5, so this variable was discarded.

As some variables had higher relative risk values than others, their punctuation was weighted in this manner:
- `age`: 1 point.
- `last_month_salary`: 1 point.
- `number_dependents`: 1 point.
- `total_loan`: 1 point.
- `more_90_days_overdue`: 3 points.
- `using_lines_not_secured_personal_assets`: 2 points.

Accordingly, the possible scores ranged between 0 and 9.

### Logistic Regression Model
To compare the classification done with the relative risk per quartile/category, a logistic regression was implemented in Python. This algorithm considered the same variables as in the previous section, including the `debt_ratio` column. The training set was composed of 80% of the data and the test set included the 20% remaining.

## Results
### Relative Risk Model

### Logistic Regression Model

## Visualization

## Conclusions and Recommendations

## References
Tenny S, Hoffman MR. Relative Risk. (Updated 2023 Mar 27). In: StatPearls. Treasure Island (FL): StatPearls Publishing; 2023 Jan. Available from: https://www.ncbi.nlm.nih.gov/books/NBK430824/
