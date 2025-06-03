---
layout: article
title:  Analysis on U.S. Electrical Power Infrastructure Resilience
mode: immersive
header:
  theme: dark
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover.jpg
---

#  Analysis on U.S. Electrical Power Infrastructure Resilience
*By Jiaying Chen, Minh Hoang*

## Table of Contents

- [Introduction](#introduction)
- [Data Cleaning and Exploratory Data Analysis](#data-cleaning-and-exploratory-data-analysis)
- [Assessment of Missingness](#assessment-of-missingness)
- [Hypothesis Testing](#hypothesis-testing)
- [Framing a Prediction Problem](#framing-a-prediction-problem)
- [Baseline Model](#baseline-model)
- [Final Model](#final-model)
- [Fairness Analysis](#fairness-analysis)

---

## Introduction

### Motivation

Power outages pose significant challenges to communities, economies, and critical services. While some of such cases are unavoidable due to naturally occuring weather conditions, there are many ways we can help improve the severity of the problem, even potetially prevent possible occurences. 
The resilience of electrical infrastructure—the ability to withstand and quickly recover from faults—is paramount to ensuring reliable electricity delivery in the face of natural hazards, equipment failures, and other disruptions. 

Our project focuses on an assessment of potential correlation between utility investment as a metric of quality to the power infrastructures, as well as many other features, then using this result to create a prediction model of the likelyhood of outages. Insights derived from this analysis can support utilities and policymakers in prioritizing infrastructure investments, improving maintenance practices, and enhancing outage management strategies—ultimately contributing to more resilient power systems capable of withstanding future challenges.


### Data

Our main dataset is from ScienceDirect: *[Data on major power outage events in the continental U.S.](https://www.sciencedirect.com/science/article/pii/S2352340918307182)*. Below are descriptions of the columns we used in the scope of this project. 


| **Column Name(s)**                               | **Description** |
| ------------------------------------------------ | --------------- |
| `year`, `month`, `state`                         |  The year and month in which the outage occured |
| `state`                                          |  The state in which the outage occurred |
| `climate_region`                                 | U.S. Climate regions as specified by National Centers for Environmental Information. There are 9 total climate regions, `East North Central`, `Central`, `South`, `Southeast`, `Northwest`, `Southwest`, `Northeast`, `West North Central`, and `West`.  |
|  `climate_cat`, `anomaly_level`                  | Indicators of El Niño/La Niña climate episodes. `anomaly_level` gives the ONI sea surface temperature deviation (3-month average), while `climate_cat` classifies the episode as `Warm`, `Cold`, or `Normal` based on ONI thresholds (±0.5 °C). |
| `cause_cat`, `cause_detail`                      | Cause of the outage, split into 7 categories.  `cause_detail` documents more elaborated description of the cause.               |
| `duration`                                       | Duration of outage in minutes.                 |
| `pc_realgsp_state`, `pi_util_of_usa`             | Regional economic characteristics. `pc_realgsp_state`:   Per capita real gross state product (GSP) in the U.S. state (measured in 2009 chained U.S. dollars. `pi_util_of_usa`:  State utility sector׳s income (earnings) as a percentage of the total earnings of the U.S. utility sector׳s income (in %)             |
| `population`                                     |  Population in the U.S. state in a year               |
| `pop_pct_urban`, `pop_pct_uc`                    | Regional land use characteristics.   	Percentage of the total population of the U.S. state represented by the urban population (in %), and represented by the urban clusters.             |
| `popden_urban`, `popden_uc`, `popden_rural`      |  Population density of the urban areas (persons per square mile), population density of the urban clusters (persons per square mile), and population density of the rural areas (persons per square mile).                |
| `area_pct_urban`, `area_pct_uc`                  |  Land area data. Percentage of the land area of the U.S. state represented by the land area of the urban areas and urban clusters (in %) |


Besides the main dataset, we also utilized the *[U.S. Energy Information Administration's Annual Electric Power Industry Report (EIA-861)](https://www.eia.gov/electricity/data/eia861/)*. Specifically, we extracted the `TOTAL` columns in `file2`. Here's the official description of this file:
> File2 contains information on retail revenue, sales, and customer counts, by State and class of service (including the Transportation sector, new in 2003), for each electric distribution utility, or energy service provider in all 50 States, the District of Columbia, the Dominion of Puerto Rico, and the Territories of American Samoa, Guam, and the Virgin Islands.
 
| **Column Name(s)**                               | **Description** |
| ------------------------------------------------ | --------------- |
| Revenues  (Thousand dollars) | Total revenue from electricity sales, reported in thousands of U.S. dollars, by utility and state. |
| Sales  (Megawatthours) | Total amount of electricity sold, measured in megawatt-hours (MWh), by utility and service class. |
| Customers Count | Number of end-use customers served by the utility, segmented by state and class of service. |

---

## Data Cleaning and Exploratory Data Analysis

*Describe how you cleaned your data and any exploratory analyses you performed to understand it better.*

---

## Assessment of Missingness

*Discuss missing data in your dataset, patterns of missingness, and how you handled them.*

---

## Hypothesis Testing

*Explain the hypotheses you tested and the results of these tests.*

---

## Framing a Prediction Problem

*Describe how you framed your problem as a prediction task, including defining inputs and outputs.*

---

## Baseline Model

*Present a simple baseline model and its performance.*

---

## Final Model

*Describe your final model, how it was trained and tuned, and its performance compared to the baseline.*

---

## Fairness Analysis

*Analyze the fairness of your model, any biases detected, and potential implications.*

---



# Contributors 

Jiaying Chen| [github](https://github.com/rcwoshimao) | [linkedin](https://www.linkedin.com/in/jiaying-chen01/)  
Minh Hoang | [github](https://github.com/thekingofrice) | [linkedin](https://www.linkedin.com/in/duc-minh-hoang-711029296/)

