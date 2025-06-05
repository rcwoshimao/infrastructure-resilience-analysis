---
layout: article
title: How Electric Generation Infrastructure and Fuel Diversity Influence Frid Resilience Across U.S. States
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

# Introduction

### Motivation

Power outages pose significant challenges to communities, economies, and critical services. While some of such cases are unavoidable due to naturally occuring weather conditions, there are many ways we can help improve the severity of the problem, even potetially prevent possible occurences. 
The resilience of electrical infrastructure—the ability to withstand and quickly recover from faults—is paramount to ensuring reliable electricity delivery in the face of natural hazards, equipment failures, and other disruptions. 

While customer demand, pricing, and regional weather events are commonly studied in power outage analysis, much less attention is paid to how the **supply-side** characteristics of a state's power grid—including generation capacity, fuel mix, and generator diversity—contribute to its resilience. A robust grid isn't just about weather-proofing—it may also reflect investments in flexible infrastructure, diversified energy sources, and state-level energy planning.

### Project Objective
We investigate the relationship between generation capacity, fuel source diversity, and actual electricity generation (via EIA-860 and EIA-923) with outage frequency, duration, and impact (via DOE outage data). In this project, we examine a few things: 
- Basic examination of outage data
- Do states with more diverse energy portfolios experiences fewer or shorter outages? 
- Are certain fuel sourses associated with better resilience? 
- Creating a prediction model utilizing features described above to predict the duration of an outage. 

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


Besides the main dataset, we also utilized the *[EIA-860 Annual Electric Generator Report](https://www.eia.gov/electricity/data/eia860/)*, and the *[EIA-923 Power Plant Operations Report](https://www.eia.gov/electricity/data/eia923/)*. Descriptions of meaningful columns are listed below.  
>The survey Form EIA-860 collects generator-level specific information about existing and planned generators and associated environmental equipment at electric power plants with 1 megawatt or greater of combined nameplate capacity. Summary level data can be found in the Electric Power Annual.

<u>💡  The columns have been renamed to `['year', 'state', 'producer type', 'fuel source', 'generators', 'facilities', 'nameplate_capacity', 'summer_capacity']` from the original dataframe, in the original order, for easier access.</u>

>The survey Form EIA-923 collects detailed electric power data -- monthly and annually -- on electricity generation, fuel consumption, fossil fuel stocks, and receipts at the power plant and prime mover level.

<u>💡  The columns have been renamed to `['year', 'state', 'producer_type', 'fuel_source', 'generation_mwh']` from the original dataframe, in the original order, for easier access.</u>


| **EIA-860 Annual Electric Generator Report: Columns**                         | **Description**                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| **`producer_type`**                  | Category of electricity producer (e.g., Utility, Industrial, Commercial).                |
| **`fuel source`**                    | Primary energy source used by the generator (e.g., Coal, Natural Gas, Solar).            |
| **`generators`**                     | Number of generator units (may contain missing data).                   |
| **`facilities`**                     | Name or count of facilities associated with generation (may contain missing data).       |
| **`nameplate_capacity`** | Maximum output capacity of a generator under ideal conditions, in megawatts (MW).        |
| **`summer_capacity'`**    | Expected maximum output of a generator during summer peak conditions, in megawatts (MW). |

| **EIA-923 Power Plant Operations Report: Columns**                         | **Description**                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| **`producer_type`**                | Same as EIA-860 **`producer_type`**. |
| **`fuel_source`** | Same as EIA-860 **`fuel source`**. |
| **`generation_mwh`** | Actual amount of output of generator in megawatts. |



# Data Cleaning and Exploratory Data Analysis
## Data Cleaning
### "outage" dataframe 
**Data cleaning**   
We have first renamed our columns for better readability.    
Since we will be focusing on assessing infrastructure resilience by state based on investment and other economic factors, we have dropped many climate or sales related columns, including outage-specific information that correlate to duration, climate conditiojns, demand-side data like total price, sales and customers, and state economy related columns. The remaining columns are listed here. 

```python 
Index(['year', 'state', 'climate_region', 'anomaly_level', 'climate_cat',
       'cause_cat', 'cause_detail', 'duration', 'pc_realgsp_state',
       'pi_util_of_usa', 'population', 'pop_pct_urban', 'pop_pct_uc',
       'popden_urban', 'popden_uc', 'popden_rural', 'area_pct_urban',
       'area_pct_uc', 'yearly_outage_count_bystate',
       'yearly_avg_duration_bystate'],
      dtype='object')
```

Some other small changes we made include: 
- Replacing `state` column with `posta_code`, to be consistent with external dataframe. 
- Dropping columns with `NaN` state and years, since those are crucial to our analysis. 
- Converted numerical columns to numerical, like `duration`, and cleaned the text of `cause_detail` for EDA later.    
**Feature Engineering**   
We created new columns for **state specific total outage counts**, and **annual state-wise average outage duration of outage**. 
The head of the final dataframe can be seen here. 

<!-- Dataframe starts -->
<div>
  <style scoped>
      .dataframe tbody tr th:only-of-type {
          vertical-align: middle;
      }

      .dataframe tbody tr th {
          vertical-align: top;
      }

      .dataframe thead th {
          text-align: right;
      }
  </style>
  <table border="1" class="dataframe">
    <thead>
      <tr style="text-align: right;">
        <th></th>
        <th>year</th>
        <th>state</th>
        <th>climate_region</th>
        <th>anomaly_level</th>
        <th>climate_cat</th>
        <th>cause_cat</th>
        <th>cause_detail</th>
        <th>duration</th>
        <th>pc_realgsp_state</th>
        <th>pi_util_of_usa</th>
        <th>population</th>
        <th>pop_pct_urban</th>
        <th>pop_pct_uc</th>
        <th>popden_urban</th>
        <th>popden_uc</th>
        <th>popden_rural</th>
        <th>area_pct_urban</th>
        <th>area_pct_uc</th>
        <th>yearly_outage_count_bystate</th>
        <th>yearly_avg_duration_bystate</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <th>0</th>
        <td>2011</td>
        <td>MN</td>
        <td>East North Central</td>
        <td>-0.3</td>
        <td>normal</td>
        <td>severe weather</td>
        <td>NaN</td>
        <td>3060.0</td>
        <td>51268</td>
        <td>2.2</td>
        <td>5348119</td>
        <td>73.27</td>
        <td>15.28</td>
        <td>2279.0</td>
        <td>1700.5</td>
        <td>18.2</td>
        <td>2.14</td>
        <td>0.6</td>
        <td>3</td>
        <td>1460.666667</td>
      </tr>
      <tr>
        <th>1</th>
        <td>2014</td>
        <td>MN</td>
        <td>East North Central</td>
        <td>-0.1</td>
        <td>normal</td>
        <td>intentional attack</td>
        <td>vandalism</td>
        <td>1.0</td>
        <td>53499</td>
        <td>2.2</td>
        <td>5457125</td>
        <td>73.27</td>
        <td>15.28</td>
        <td>2279.0</td>
        <td>1700.5</td>
        <td>18.2</td>
        <td>2.14</td>
        <td>0.6</td>
        <td>2</td>
        <td>30.500000</td>
      </tr>
      <tr>
        <th>2</th>
        <td>2010</td>
        <td>MN</td>
        <td>East North Central</td>
        <td>-1.5</td>
        <td>cold</td>
        <td>severe weather</td>
        <td>heavy wind</td>
        <td>3000.0</td>
        <td>50447</td>
        <td>2.1</td>
        <td>5310903</td>
        <td>73.27</td>
        <td>15.28</td>
        <td>2279.0</td>
        <td>1700.5</td>
        <td>18.2</td>
        <td>2.14</td>
        <td>0.6</td>
        <td>3</td>
        <td>2610.000000</td>
      </tr>
      <tr>
        <th>3</th>
        <td>2012</td>
        <td>MN</td>
        <td>East North Central</td>
        <td>-0.1</td>
        <td>normal</td>
        <td>severe weather</td>
        <td>thunderstorm</td>
        <td>2550.0</td>
        <td>51598</td>
        <td>2.2</td>
        <td>5380443</td>
        <td>73.27</td>
        <td>15.28</td>
        <td>2279.0</td>
        <td>1700.5</td>
        <td>18.2</td>
        <td>2.14</td>
        <td>0.6</td>
        <td>1</td>
        <td>2550.000000</td>
      </tr>
      <tr>
        <th>4</th>
        <td>2015</td>
        <td>MN</td>
        <td>East North Central</td>
        <td>1.2</td>
        <td>warm</td>
        <td>severe weather</td>
        <td>NaN</td>
        <td>1740.0</td>
        <td>54431</td>
        <td>2.2</td>
        <td>5489594</td>
        <td>73.27</td>
        <td>15.28</td>
        <td>2279.0</td>
        <td>1700.5</td>
        <td>18.2</td>
        <td>2.14</td>
        <td>0.6</td>
        <td>2</td>
        <td>947.500000</td>
      </tr>
    </tbody>
  </table>
</div>

<!-- Dataframe ends -->

### "capacity" and "generated" dataframe 
We were originally going to using capacity per capita, but this didnt work because there are too many missing population values in state-year population pairs. After doing some research, we decided to engineer the features `capacity_fuel_diversity_index` and `generation_fuel_diversity_index`, by converting the info in our dataframe into **Shannon Entropy**. 

> Shannon entropy is a measure from information theory that quantifies the uncertainty or diversity in a dataset. In the context of energy:
> Higher Entropy: Indicates a more diverse energy mix, with energy production or capacity spread across multiple fuel sources.
> Lower Entropy: Suggests reliance on fewer fuel sources, indicating less diversity.   

While both capacity and generation metrics use Shannon entropy, they capture different aspects:
Capacity Entropy: Reflects the potential diversity based on installed infrastructure. It indicates how prepared a state is to utilize various energy sources.   

Generation Entropy: Represents the actual diversity in energy production. It shows how the energy mix is utilized in practice.    

Research suggests that energy systems with higher fuel diversity are more resilient to disruptions. A diverse energy mix can mitigate the impact of outages in specific fuel sources.  
For instance, a study from the University of Texas at Austin analyzed diversity trends in U.S. electricity generation using Shannon entropy and other indices, highlighting the importance of a balanced energy mix for system resilience.   

We believe including both scores will allow us to have insights into the discrepancies between **potential** and **actual** energy diversity, which may affect resilience and outage durations.  

The head of the cleaned and engineered dataframes can be seen below.

**`capacity_entropy`**
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>state</th>
      <th>year</th>
      <th>capacity_fuel_diversity_index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AK</td>
      <td>2000</td>
      <td>0.970639</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AK</td>
      <td>2001</td>
      <td>0.969751</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AK</td>
      <td>2002</td>
      <td>0.955367</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AK</td>
      <td>2003</td>
      <td>0.882028</td>
    </tr>
    <tr>
      <th>4</th>
      <td>AK</td>
      <td>2004</td>
      <td>0.874298</td>
    </tr>
  </tbody>
</table>
</div>

**`generation_entropy`**
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>state</th>
      <th>year</th>
      <th>generation_fuel_diversity_index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AK</td>
      <td>2000</td>
      <td>1.022950</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AK</td>
      <td>2001</td>
      <td>1.020200</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AK</td>
      <td>2002</td>
      <td>1.016209</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AK</td>
      <td>2003</td>
      <td>0.917580</td>
    </tr>
    <tr>
      <th>4</th>
      <td>AK</td>
      <td>2004</td>
      <td>0.911863</td>
    </tr>
  </tbody>
</table>
</div>

### Merging engineered features into outages
After merning, the head of our final dataframe `outage` can be seen below.

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>state</th>
      <th>climate_region</th>
      <th>anomaly_level</th>
      <th>climate_cat</th>
      <th>cause_cat</th>
      <th>cause_detail</th>
      <th>duration</th>
      <th>pc_realgsp_state</th>
      <th>pi_util_of_usa</th>
      <th>...</th>
      <th>pop_pct_uc</th>
      <th>popden_urban</th>
      <th>popden_uc</th>
      <th>popden_rural</th>
      <th>area_pct_urban</th>
      <th>area_pct_uc</th>
      <th>yearly_outage_count_bystate</th>
      <th>yearly_avg_duration_bystate</th>
      <th>capacity_fuel_diversity_index</th>
      <th>generation_fuel_diversity_index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2011</td>
      <td>MN</td>
      <td>East North Central</td>
      <td>-0.3</td>
      <td>normal</td>
      <td>severe weather</td>
      <td>NaN</td>
      <td>3060.0</td>
      <td>51268</td>
      <td>2.2</td>
      <td>...</td>
      <td>15.28</td>
      <td>2279</td>
      <td>1700.5</td>
      <td>18.2</td>
      <td>2.14</td>
      <td>0.6</td>
      <td>3</td>
      <td>1460.666667</td>
      <td>1.045207</td>
      <td>0.978170</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2014</td>
      <td>MN</td>
      <td>East North Central</td>
      <td>-0.1</td>
      <td>normal</td>
      <td>intentional attack</td>
      <td>vandalism</td>
      <td>1.0</td>
      <td>53499</td>
      <td>2.2</td>
      <td>...</td>
      <td>15.28</td>
      <td>2279</td>
      <td>1700.5</td>
      <td>18.2</td>
      <td>2.14</td>
      <td>0.6</td>
      <td>2</td>
      <td>30.500000</td>
      <td>1.065896</td>
      <td>0.999916</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2010</td>
      <td>MN</td>
      <td>East North Central</td>
      <td>-1.5</td>
      <td>cold</td>
      <td>severe weather</td>
      <td>heavy wind</td>
      <td>3000.0</td>
      <td>50447</td>
      <td>2.1</td>
      <td>...</td>
      <td>15.28</td>
      <td>2279</td>
      <td>1700.5</td>
      <td>18.2</td>
      <td>2.14</td>
      <td>0.6</td>
      <td>3</td>
      <td>2610.000000</td>
      <td>1.032858</td>
      <td>0.969911</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2012</td>
      <td>MN</td>
      <td>East North Central</td>
      <td>-0.1</td>
      <td>normal</td>
      <td>severe weather</td>
      <td>thunderstorm</td>
      <td>2550.0</td>
      <td>51598</td>
      <td>2.2</td>
      <td>...</td>
      <td>15.28</td>
      <td>2279</td>
      <td>1700.5</td>
      <td>18.2</td>
      <td>2.14</td>
      <td>0.6</td>
      <td>1</td>
      <td>2550.000000</td>
      <td>1.049795</td>
      <td>1.013572</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2015</td>
      <td>MN</td>
      <td>East North Central</td>
      <td>1.2</td>
      <td>warm</td>
      <td>severe weather</td>
      <td>NaN</td>
      <td>1740.0</td>
      <td>54431</td>
      <td>2.2</td>
      <td>...</td>
      <td>15.28</td>
      <td>2279</td>
      <td>1700.5</td>
      <td>18.2</td>
      <td>2.14</td>
      <td>0.6</td>
      <td>2</td>
      <td>947.500000</td>
      <td>1.072869</td>
      <td>1.004192</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 22 columns</p>
</div>

## Exploratory Data Analysis


---

# Assessment of Missingness

*Discuss missing data in your dataset, patterns of missingness, and how you handled them.*

---

# Hypothesis Testing

*Explain the hypotheses you tested and the results of these tests.*

---

# Framing a Prediction Problem

*Describe how you framed your problem as a prediction task, including defining inputs and outputs.*

---

# Baseline Model

*Present a simple baseline model and its performance.*

---

# Final Model

*Describe your final model, how it was trained and tuned, and its performance compared to the baseline.*

---

# Fairness Analysis

*Analyze the fairness of your model, any biases detected, and potential implications.*

---



# Contributors 

Jiaying Chen| [github](https://github.com/rcwoshimao) | [linkedin](https://www.linkedin.com/in/jiaying-chen01/)  
Minh Hoang | [github](https://github.com/thekingofrice) | [linkedin](https://www.linkedin.com/in/duc-minh-hoang-711029296/)

