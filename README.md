# Credit Score Based on the Risk of Non-Payment

![image](https://github.com/karlarochaes/credit-score-relative-risk/assets/88100992/2a6e3bc1-977a-4d6e-8c51-c2b98f2abb7a)

---

## Tabla of Contents
- [Introduction](#introduction)
- [Tools](#tools)
- [Data Processing](#data-processing)
- [Relative risk](#relative-risk)
- [Results](#results)
- [Visualization](#visualization)
- [Conclusions and Recommendations](#conclusions-and-recommendations)

## Introduction
The "Super Caja" bank is facing an increasing amount of credit requests. In this context, it is necessary to improve the credit request revision system, in order to save time-resources and reduce the risk of non-payment. To accomplish these tasks, the main goal of this project was to calculate the relative risk associated to several user variables and estimate a global score to accurately differenciate the bad payers from the good ones.

## Tools
- SQL (Google BigQuery)
- Python ([Google Colab](link))
- Looker Studio

## Data Processing
### Data Cleaning
#### Null values

#### Outliers

### New Variables

## Relative risk
The relative risk was calculated as follows:

$Relative\ risk = {Incidence\ risk\ among\ an\ exposed\ group \over Incidence\ risk\ among\ a\ non-exposed\ group}$


## Logistic Regression
To compare the classification done with the relative risk per quartile/category, a logistic regression was implemented in Python. This algorithm considered the same variables as in the previous section.

## Results
### Confusion Matrix

### Metrics

## Visualization

## Conclusions and Recommendations
