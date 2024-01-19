# Credit Score Based on the Risk of Non-Payment

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/2a6e3bc1-977a-4d6e-8c51-c2b98f2abb7a)

---

## Table of Contents
- [Introduction](#introduction)
- [Tools](#tools)
- [Data Sourcing](#data-sourcing)
- [Data Cleaning](#data-cleaning)
- [Models](#models)
- [Results](#results)
- [Visualization](#visualization)
- [Conclusions and Recommendations](#conclusions-and-recommendations)

## Introduction
The "Super Caja" bank is facing an increasing amount of credit requests. In this context, it is necessary to improve the credit request revision system, in order to save time resources and reduce the risk of non-payment. To accomplish these tasks, the main goal of this project was to calculate the relative risk associated to several user variables and estimate a global score to accurately differenciate the bad payers from the good ones.

## Tools
- SQL (Google BigQuery)
- Python (Google Colab)
- [Looker Studio](https://lookerstudio.google.com/reporting/29b10b35-77d0-4287-b4b4-f7400f0e3ff6/page/TBDnD)

## Data Sourcing
Four tables:
- **user_info** with 36000 rows and 5 columns:
  - `user_id`: unique identification number for clients.
  - `age`: client age.
  - `sex`: client sex.
  - `last_month_salary`: last month salary reported.
  - `number_dependents`: client number of dependents.
  
- **loans_outstanding** with 305335 rows and 3 columns:
  - `loan_id`: unique identification number for loans.
  - `user_id`: unique identification number for clients.
  - `loan_type`: type of loan (real_estate / other).

- **loans_detail** with 36000 rows and 6 columns:
  - `user_id`: unique identification number for clients.
  - `more_90_days_overdue`: number of times the client has been more than 90 days overdue.
  - `using_lines_not_secured_personal_assets`: how much the client is using relative to their credit limit, on lines that are not secured by personal property, such as real estate and automobiles.
  - `number_times_delayed_payment_loan_30_59_days`: number of times the client was late paying a loan (between 30 and 59 days).
  - `number_times_delayed_payment_loan_60_89_days`: number of times the client was late paying a loan (between 60 and 89 days).
  - `debt_ratio`: relationship between debts and the borrower's assets. debt ratio = debts / assets.

- **default** with 36000 rows and 2 columns:
  - `user_id`: unique identification number for clients.
  - `default_flag`: classification of defaulter clients (1 for bad-paying customers, 0 for good-paying customers).

## Data Cleaning
### Drop Columns
First of all, to guarantee non-biased results, the `sex` variable was discarded. 

Next, correlation between variables was checked. This led to discarding the `number_times_delayed_payment_loan_30_59_days` and the `number_times_delayed_payment_loan_60_89_days` variables, which had a high positive correlation with `more_90_days_overdue`.

Finally, the `loan_type` variable was split into three columns: 
- `real_estate_loan`: counts the Real Estate loans per user.
- `other_loan`: counts the other type loans per user.
- `total_loan`: counts the total loans per user.

As these variables were correlated, later in the project it was determined whether it was better to use the first two columns or only the `total_loan` column.

### Null values
The original tables contained 7199 null values in the `last_month_salary` column and 943 in the `number_dependents` column. Once it was determined that the null values were randomly located, the K-Nearest Neighbors (KNN) algorithm was used to predict and impute those missing values.

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/61168e46-a218-4e0b-9b5d-3952a14bad1f)

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/ee7a6d36-921e-4a80-8427-8dfb734a5ec7)

### Outliers
The outliers were handled capping the outliers to the maximum value using the IQR range:
- `age`: any value higher than 97 was set to 97.
- `last_month_salary`: any value higher than 15650 was set to 15650.
- `more_90_days_overdue`: any value higher than 15 was set to 15.
- `using_lines_not_secured_personal_assets`: any value higher than 1.31 was set to 1.31. Rounded to two decimals.
- `debt_ratio`: any value higher than 1 was set to 1. Rounded to two decimals.

Using this approach, the original 36000 rows were preserved.

## Models

### Relative Risk Model

#### Quartiles
To prepare the data for further analysis and find patterns per group, each variable was divided into quartiles. If the variable already contained a considerable amount of zeros, the creation of only two or three categories was preferred over the creation of quartiles. Then, the relative risk was calculated in each group.

#### Relative Risk
The relative risk was calculated as follows:

$Relative\ risk = {Incidence\ risk\ among\ an\ exposed\ group \over Incidence\ risk\ among\ a\ non-exposed\ group}$

||Event (case)|No event (control)|Total|
|--------|--------|--------|--------|
|Exposed|a|b|a+b|
|Non-exposed|c|d|c+d|

$Relative\ risk = {a/a+b \over c/c+d}$

In the context of this project, the exposure factors were the user variables (age, salary, overdue, etc.) and the event or disease was the `default_flag` column, meaning bad antecedents with credit. So, for example, in the case of the `age` variable, the calculation for the relative risk in the first quartile was stated as:

||default_flag=1|default_flag=0|Total|
|--------|--------|--------|--------|
|Q1 (21-41 years)|311|8764|9075|
|Q2, Q3, Q4 (other age ranges)|372|26553|26925|

$Relative\ risk = {311/9075 \over 372/26925} = {0.034 \over 0.013} = 2.48$  

This number means that the youngest clients are **2.48** times more likely to be bad payers than older clients. 

The corresponding calculus was estimated for each quartile or category and for each variable considered in the model. The `total_loan` column was preserved since it contained the highest relative risk value compared to the `real_estate_loan` and `other_loan` columns.

#### Credit Score
The relative risk values were transformed to binary values (dummy variables) using the following criteria:
- 1 if the relative risk was higher than 1.5
- 0 if the relative risk was equal to or lower than 1.5

None of the `debt_ratio` quartiles had a relative risk value higher than 1.5, so this variable didn't add any points.

### Logistic Regression Model
To compare the classification done with the relative risk per quartile/category, a logistic regression was implemented in Python. This algorithm considered the same variables as in the previous section, including the `debt_ratio` column. The training set was composed of 80% of the data and the test set included the remaining 20%.

### Evaluation of Model Performance
The two models were evaluated using the confusion matrix and four performance metrics related. The confusion matrix compares the real values to the predicted values in each category (good payers and bad payers):

|real/predicted|good payer|bad payer|
|--------|--------|--------|
|good payer|True Negatives (TN)|False Positives (FP)|
|bad payer|False Negatives (FN)|True Positives (TP)|

Performance metrics: 
- accuracy: proportion of correctly classified instances among the total instances. It is a measure of overall correctness.

  $accuracy = {TP+TN \over TP+TN+FP+FN}$

- recall: ability of a model to capture all the relevant instances of a particular class. It is the ratio of correctly predicted positive observations to the total actual positives.

  $recall = {TP \over TP+FN}$
  
- precision: quantifies the accuracy of positive predictions made by the model. It is the ratio of correctly predicted positive observations to the total predicted positives.

  $precision = {TP \over TP+FP}$
  
- F1 score: harmonic mean of precision and recall. It provides a balance between precision and recall, especially when there is an uneven class distribution.

  $F1\ score = {2 * Precision * Recall \over Precision+Recall}$

## Results
### Relative Risk Model

An initial non-weighted model was found to perform poorly. Next, this model was improved assigning a weight for each point. As some variables had higher relative risk values than others, their punctuation was weighted in this manner:
- `more_90_days_overdue`: 3 points.
- `using_lines_not_secured_personal_assets`: 2 points.
- `total_loan`: 1 point.
- `age`: 1 point.
- `last_month_salary`: 1 point.
- `number_dependents`: 1 point.

Accordingly, the possible scores ranged between 0 and 9.

To find the best cutoff point to classify bad payers and good payers, the model was evaluated using the performance metrics and the confusion matrix previously described.

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/0b7e9ad5-94b3-4f55-86bc-ec28a628b7ee)

Given that bad payers were the principal category of interest in this analysis, the recall value was prioritized over the rest of the performance metrics. Therefore, a cutoff point of 5 was chosen. Although cutoff points >=6 correspond to higher F1 score and other metric values, these also correspond to lower recall values.

A cutoff point of 5 produced the following confusion matrix:

|real/predicted|good payer|bad payer|
|--------|--------|--------|
|good payer|32844|2473|
|bad payer|77|606|

### Logistic Regression Model
The confusion matrix for this model was configured as follows:

|real/predicted|good payer|bad payer|
|--------|--------|--------|
|good payer|6753|302|
|bad payer|10|135|

Then, the performance metrics of the two models were compared:

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/d0de6a70-d835-4fca-90ed-107a8010af08)

As can be seen, the logistic regression model performed better than the relative risk model in all metrics. Nevertheless, the difference was small for the accuracy and recall metrics.

In summary, although the logistic regression model classified good and bad payers better than the relative risk model, the information of both models may be used complementarily.

## Visualization
The final dashboard can be consulted in [this link](https://lookerstudio.google.com/reporting/29b10b35-77d0-4287-b4b4-f7400f0e3ff6).

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/a4cb3370-4392-41c6-a38b-f071f6b4607a)


## Conclusions and Recommendations
- The most important variables are those related to the financial information of the client (number of times overdue, use of lines not secured with personal assets, and total loans). Accordingly, these should be the primary characteristics to identify probable defaulters.
- A total score equal to or greater than 5 allows us to identify probable defaulter clients. Consequently, this would help reduce the risk of non-payment in the business.
- The two models presented can be used complementarily. The relative risk model focuses on which specific group or range per variable is most likely to be bad payer, and the logistic regression model improves the classification made to accurately make a decision about credit requests.

## References
Tenny, S. & Hoffman, M.R. Relative Risk. (2023). In: StatPearls. Treasure Island (FL): StatPearls Publishing. Available from: https://www.ncbi.nlm.nih.gov/books/NBK430824/
