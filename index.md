---
layout: article
title: How Electric Generation Infrastructure and Fuel Diversity Influence Grid Resilience Across U.S. States
mode: immersive
mathjax: true
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

*By Jiaying Chen, Duc Minh Hoang*

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
We investigate the relationship between generation capacity, fuel source diversity, and actual electricity generation (via EIA-860 and EIA-923) with outage frequency, duration, and impact (via DOE outage data). In this project, we: 
- Perform basic examination of outage data
- Explored the question "Do states with more diverse energy portfolios experiences fewer or shorter outages?" 
- ...And Created a prediction model utilizing features described above to predict the duration of an outage. 

Understanding the causes and characteristics of major power outages is more critical than ever in today’s climate-challenged and energy-dependent world. Our dataset combines detailed records of U.S. outage events with regional economic, environmental, and energy generation data, offering a rare opportunity to explore how infrastructure resilience varies across states.
By building a model to predict average outage duration based on interpretable features that are known ahead of time, our project aims to offer a small but practical step toward using data to better understand grid reliability.

### Data

Our main dataset is from ScienceDirect: *[Data on major power outage events in the continental U.S.](https://www.sciencedirect.com/science/article/pii/S2352340918307182)*.    

**The raw dataset has 1534 rows and 56 columns.**    
Below are descriptions of the columns we used in the scope of this project. 


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

**This raw dataset has 62282 rows and 5 columns.**    
💡  The columns have been renamed to `['year', 'state', 'producer type', 'fuel source', 'generators', 'facilities', 'nameplate_capacity', 'summer_capacity']` from the original dataframe, in the original order, for easier access.



| **EIA-860 Annual Electric Generator Report: Columns**                         | **Description**                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| **`producer_type`**                  | Category of electricity producer (e.g., Utility, Industrial, Commercial).                |
| **`fuel source`**                    | Primary energy source used by the generator (e.g., Coal, Natural Gas, Solar).            |
| **`generators`**                     | Number of generator units (may contain missing data).                   |
| **`facilities`**                     | Name or count of facilities associated with generation (may contain missing data).       |
| **`nameplate_capacity`** | Maximum output capacity of a generator under ideal conditions, in megawatts (MW).        |
| **`summer_capacity'`**    | Expected maximum output of a generator during summer peak conditions, in megawatts (MW). |

>The survey Form EIA-923 collects detailed electric power data -- monthly and annually -- on electricity generation, fuel consumption, fossil fuel stocks, and receipts at the power plant and prime mover level.

**This raw dataset has 54002 rows and 8 columns.**      
💡  The columns have been renamed to `['year', 'state', 'producer_type', 'fuel_source', 'generation_mwh']` from the original dataframe, in the original order, for easier access.


| **EIA-923 Power Plant Operations Report: Columns**                         | **Description**                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| **`producer_type`**                | Same as EIA-860 **`producer_type`**. |
| **`fuel_source`** | Same as EIA-860 **`fuel source`**. |
| **`generation_mwh`** | Actual amount of output of generator in megawatts. |



# Data Cleaning and Exploratory Data Analysis
## Data Cleaning
### "outage" dataframe 
#### Data cleaning 

We began by renaming columns for improved readability and dropped many columns related to climate, sales, and outage-specific details that were less relevant to assessing infrastructure resilience from an economic and investment perspective. For example, columns related to outage duration drivers, demand-side metrics like total price or customers, and some state economy indicators were removed. The resulting columns retained include key variables such as year, state, climate region, outage duration, economic indicators, population metrics, and state-level outage summaries. The remaining columns are listed here. 

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
- Dropping columns with `NaN` state and years to maintain data integrity.
- Converted numerical columns to numerical, like `duration`, and cleaned the text of `cause_detail` for EDA later.    

#### Feature Engineering 
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

The raw capacity and generation datasets contained several inconsistencies, such as population data missing for many state-year pairs, capacity and generation values recorded as strings with commas, and fuel source categories varying in aggregation. We filtered data to the years 2000 to 2016 to ensure completeness and consistency. Numeric columns had commas removed and were converted to numeric types to enable calculations. For capacity, we kept only rows where fuel source was “All Sources” to represent total installed capacity, and for generation, we excluded rows labeled “Total” to focus on individual fuel sources. Missing or invalid values in generation were dropped. These cleaning steps ensured that the data was accurate and consistent for calculating diversity indices. 

We were originally going to use capacity per capita as a feature, but this didnt work because there are too many missing population values in state-year population pairs. After doing some research, we decided to engineer the features `capacity_fuel_diversity_index` and `generation_fuel_diversity_index`, by converting the info in our dataframe into **Shannon Entropy**. 

> Shannon entropy is a measure from information theory that quantifies the uncertainty or diversity in a dataset. In the context of energy:   
> **Higher Entropy**: Indicates a more diverse energy mix, with energy production or capacity spread across multiple fuel sources.   
> **Lower Entropy**: Suggests reliance on fewer fuel sources, indicating less diversity.    

<div style="text-align: center;">
  <img src="https://miro.medium.com/v2/resize:fit:1400/1*6i3r04xpc-NG_K0LYKU7Sg.png" alt="Shannon Entropy Formula" width="300">
</div>


While both capacity and generation metrics use Shannon entropy, they capture different aspects. Capacity Entropy reflects the potential diversity based on installed infrastructure; It indicates how prepared a state is to utilize various energy sources. Generation Entropy represents the actual diversity in energy production. It shows how the energy mix is utilized in practice.    

Research suggests that energy systems with higher fuel diversity are more resilient to disruptions. A diverse energy mix can mitigate the impact of outages in specific fuel sources. For instance, a study from the University of Texas at Austin analyzed diversity trends in U.S. electricity generation using Shannon entropy and other indices, highlighting the importance of a balanced energy mix for system resilience.   

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
<table border="0" class="dataframe">
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
<table border="0" class="dataframe">
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
### Univariate Analysis 

#### Outage caused by climate region

<iframe
  src="assets/plots/climate_region_cntplot.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

From the plot, we can see that the Northeast climate region has the most number of outages, followed by South, while West North Cental and Southwest sit at the bottom. This can be influenced by several factors: 
- Population density. The South climate region houses approximately 37% of the nation's population. High urbanization can contribute to a higher frequency of outages. 
- The South and Northeast are more susceptible to severe weather events like hurricanes, thunderstorms, and snostorms, which can disrupt power infrastructure. 

#### Log duration of Outages histogram
<iframe
  src="assets/plots/log_duration.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

This distribution histogram highlights the varying nature of outage causes and their corresponding restoration times, from brief, automatically resolved incidents to prolonged outages due to significant disruptions. We made some inferences of possibel causes of outages: 

- 0 to 1 Minute: These brief outages could be attributed to transient faults, such as momentary disruptions cleared by automatic systems.

- 1 Minute to 4–7 Hours: This gradual increase may reflect outages caused by more significant issues like equipment failures or localized weather events requiring manual intervention.

- 1 to 2 Days: The peak in this range suggests the impact of major events, such as severe storms or natural disasters, leading to prolonged restoration times.

- 7 to 11 Weeks: The decline here indicates that extremely long outages are rare, possibly resulting from catastrophic events or extensive infrastructure damage.

#### Cause detail word cloud 
<iframe
  src="assets/plots/cause_wordcloud.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

Based on the cleaned `cause_detail` column, a wordcloud was made. 
- Vandalism and Sabotage are the most prominent words; this suggests human-induced disruptions, rather than natrual disasters, is a significant concern. 
- Outside of human interference, thunderstorms, storms, hurricanes and snowice point to natrual events as still major contributors to outages. 
- Words like "trip" and "interruption" maye refer to system operation issues or planed maintainance activities. 
Besides the above, the word "heavy" also accidencally intruded as a prominent word; We believe this could be an incomplete descriptor, e.g. "heavy rain" or "heavy snow". 

### Bivariate Analysis 
#### Boxplot of anomaly levels by state
<iframe
  src="assets/plots/anomaly_by_state.html"
  width="1000"
  height="400"
  frameborder="0"
></iframe>

Some patterns we observed: 
- South Dakota (SD) and Montana (MT) have the fewest data points, possibly due to lower population densities or less comprehensive reporting mechanisms. 
- Kansas (KS) has a very obvious pronounced lower tail, suggesting occasional significant negative anomalies. 
- States like Utah (UT) and California (CA) have extened upper tails, indicating sporadic high anomaly events. 
  - 💡 California's upper tail might be influenced by events like the [2019 Public Safety Power Shutoffs](https://en.wikipedia.org/wiki/2019_California_power_shutoffs) implemented to prevent wild fires! 
- Michigan (MI) stands our with numeraous outliers at the top, reflecting a wide range of anomaly levels. 
  - 💡 This might be attributed to a combination of urban and rural areas experiencing varyin outage causes, from equipment failures to weather related incidents. 

#### Climate Region and Cause Category Pie Chart
<iframe
  src="assets/plots/climate_cause_pie.html"
  width="1000"
  height="550"
  frameborder="0"
  display="block"
></iframe>

The northeast, south and west occupy the largest segments, aligning with their higher population densities and infrastructural complexities. Severe weather is the most primary cause in most climate regions, followed by intentional attack. 
- In the south climate region, public appeal is the second most prominent cause. A public appeal refers to requests made by utility companies or grid operators urging the public to voluntarily reduce electricity consumption during times of high demand or limited supply. This is used to prevent overloading the grid and avert potential outages. 


 > 💡 This Sunburt chart idea was inspired by my friend, Mohak Akul Prakash's DSC 80 Project. Thank you, Mohak!   
 >[Checkout his project here.](https://earthwolves.github.io/power-outage-cause-prediction/)

#### Average Yearly Outage Count by Stare Heatmap
<iframe
  src="assets/plots/count_heatmap.html"
  width="800"
  height="800"
  frameborder="0"
></iframe>

In this State v.s. Years heatmap, we can observe a temporal trend of general decline in outages over the years. This can suggest improvements in infrastructure, better maintenance practices, and enhanced grid management. 

- States like California, Texas, Washington, Michigan and New York had higher outage counts, likely due to their large populations and complex power systems. 
- The 2004 spike in Florida (17 outages compared to an average of approxinately 5 from other states) can likely be due to Hurricane Charley, Hurrican Frances, Hurrican Ivan and Hurrican Jeanne. (Based on [Power Outages Report in Florida](https://poweroutage.report/fl?utm_source=chatgpt.com)).

#### Average Outage Duration: Year-month and Year 
<iframe
  src="assets/plots/avg_outage_duration.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

There are some condensed number of spike of power outage count from 2003 to 2005.    
- The most prominent spike we can observe on this graph might be caused by [the 2003 Halloween Solar Storms](https://en.wikipedia.org/wiki/2003_Halloween_solar_storms).    
- On January 27, 2009, an ice storm hit Kentucky and in Southern Indiana knocking out power to about 769,000. As of February 15, about 12,000 were still without power from this storm. [This information can be found on wikipedia.](https://en.wikipedia.org/wiki/List_of_major_power_outages#:~:text=system%20to%20trip.-,2009,without%20power%20from%20this%20storm.)

### Interesting Aggregate

We constructed a pivot table to examine average outage durations segmented by cause categories and climate categories. Given the complexity of interpreting raw numeric values, we visualized this data as a heatmap to better reveal underlying patterns and relationships. 
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
<table border="0" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>climate_cat</th>
      <th>cold</th>
      <th>normal</th>
      <th>warm</th>
    </tr>
    <tr>
      <th>cause_cat</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>equipment failure</th>
      <td>308.235294</td>
      <td>3201.428571</td>
      <td>505.000000</td>
    </tr>
    <tr>
      <th>fuel supply emergency</th>
      <td>17433.000000</td>
      <td>7658.823529</td>
      <td>22799.666667</td>
    </tr>
    <tr>
      <th>intentional attack</th>
      <td>497.282051</td>
      <td>426.817778</td>
      <td>312.557377</td>
    </tr>
    <tr>
      <th>islanding</th>
      <td>259.266667</td>
      <td>142.176471</td>
      <td>209.833333</td>
    </tr>
    <tr>
      <th>public appeal</th>
      <td>2125.909091</td>
      <td>1376.529412</td>
      <td>596.230769</td>
    </tr>
    <tr>
      <th>severe weather</th>
      <td>3288.025316</td>
      <td>4070.401709</td>
      <td>4458.697368</td>
    </tr>
    <tr>
      <th>system operability disruption</th>
      <td>601.861111</td>
      <td>953.589286</td>
      <td>478.200000</td>
    </tr>
    <tr>
      <th>Overall</th>
      <td>2659.748918</td>
      <td>2537.369505</td>
      <td>2828.654804</td>
    </tr>
  </tbody>
</table>
</div>

Since it is hard to interpret the numeric feature above, we thought of creating a heatmap based on the data given, and we can see some interesting results.    
<iframe
  src="assets/plots/pivot_fig.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

Several notable observations emerge from this visualization:   
- Fuel Supply Emergencies are associated with the longest outage durations across all climate regions. Among the three climate categories, warm climates appear to be most affected by fuel supply emergencies, which may be driven by a higher frequency of such incidents in these regions.

- Severe Weather is the next most impactful cause category; However, the associated outage durations are substantially shorter, averaging approximately 4,000 to 5,000 minutes. This contrasts sharply with fuel supply emergencies, which exhibit average durations ranging between 17,000 and 22,000 minutes.   

These insights suggest that while severe weather is a significant cause of outages, fuel supply emergencies have a more prolonged and potentially disruptive impact on utility reliability.   




---

# Assessment of Missingness

Upon inspection, we found that from our cleaned dataframe, the columns with the most amount of missing data are `cause_detail`, `duration`, `popden_rural`, `popden_uc`, and`anomaly_level`. 

### NMAR Assessments
#### Addressing Missingness of Cause Detail
The missingness of the **cause_detailed** col is most likely NMAR since the details might give sensitive information about an individual. Usually when an outage occurs, we can expect to see a recording or failsafe that indicates what was broken. That said, if the outage occurs in under-resourced areas then there is a much higher chance that it is not recorded at all due to lack of infrastructure.

#### Addressing Missingness of Duration
On the other hand, we have reason believe that **duration** not NMAR since even though we can make a case that if a duration is too low or too high it might not be recorded. Why we believe that reasoning is flawed is because the **duration** column has 78 entries that only have 0 duration. Since, duration is non negative and the max duration is rather high, we can say that this missingness cannot be determined by the value within the column itself. If anything, if an outage lasts very long then there is more reason that it should be recorded for record keeping. So it is not NMAR.

<!-- We believe it may be MAR because the duration of an outage can vary based on how the population is concentrated, the cause detail (typhoon or natural disasters more likely to cause much more damage to the infrastructure making it less likely that the equipment to collect the data may be tampered with), month is there is a pattern of months that specifically has many outages that l -->

#### Addressing Missingness of Climate Category and Region
It doesn't make much sense to label this as NMAR because all geographical regions in the country are labeled with their respective category and region, so it is unlikely that an outtage would happen in an area that cannot be classified as one of them. Thus, it cannot be that the missingness is dependent on the value of the entry within the coclumn itself. 

#### Addressing Missingness of popden_uc, popden_rural
There is no reason to believe that these columns are NMAR since the population level, urban or not, is highly dependent on the state and its population, both of which are features in our data. 

### MAR Testing
#### Set up 
Cause detail is the column with the most missingness out of our cleaned dataset. We find it intriguing to examine the missingness dependency of this column. 

We performed **permutaton testing** to compare distributions of other variables conditional on the presence or absense `cause_detail`.
- For numeric variables: We used mean differences as the test statistic. 
- For categorical columns, we used TVD.

#### Result 
We found statistically significant evidence (at α = 0.05) that the missingness of `cause_detail` is dependent on 11 of 21 columns, including:

Demographic and density features: `population`, `pop_pct_urban`, `popden_urban`, `popden_uc`, `popden_rural`

Infrastructure and geographic metrics:` pi_util_of_usa`, `area_pct_urban`, `area_pct_uc`

Outage-related metrics: `yearly_outage_count_bystate`, `yearly_avg_duration_bystate`

This suggests that missing `cause_detail` values are not missing completely at random and are instead likely MAR — i.e., they depend on observable variables.


<iframe
  src="assets/plots/mar_pc_realgsp.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>
<iframe
  src="assets/plots/pc_realgsp_distribution.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

<iframe
  src="assets/plots/mar_yearly_avg_duration.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>
<iframe
  src="assets/plots/yearly_avg_duration_distribution.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>


---

# Hypothesis Testing
### Set up 

We investigate whether states with more diverse energy sources tend to experience less severe power outages. Specifically, we ask:

> Is there evidence that higher fuel diversity is associated with shorter average power outage durations?

This question ties together the engineered features described in previous steps:
- Fuel Diversity Score: Two measures of the distribution of different power sources (e.g., coal, gas, hydro) used in each state, calculate given the external datasets above. 
   - Capacity Entropy: Reflects infrastructure potential.
   - Generation Entropy: Reflects actual usage patterns.
- Average Outage Duration by state: A proxy for utility robustness and outage severity for each state. 

### Hypothesis 
We split states into high and low diversity groups based on the median fuel diversity index (separately for capacity and generation).

**Null Hypothesis:**   
States with high and low fuel diversity have the same average outage durations.    
**Alternative Hypothesis:**   
States with higher fuel diversity have shorter average outage durations.

### Methodology
Test statistic: We picked **differences in means as our test statistic**, since our hypotheis is directional - We want to see if high fuel diversity can be related to shorter outage duration. This test statistic is simple and easy to interpret. 

**Significance Level**: 0.05. A common signifiance level used for most hypotheis tests.

### Results 
#### Capacity Fuel Diversity Index 
**Observed Difference**: 600.452   
**P-value**: 0.500

<iframe
  src="assets/plots/ht_capacity.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

#### Generated Fuel Diversity Index 
**Observed Difference**: 419.587   
**P value**: 0.492

<iframe
  src="assets/plots/ht_generated.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

### Interpretation and Conclusion
In both tests, the p-values are far above the 0.05 significance threshold, meaning we fail to reject the null hypothesis. In other words, we do not find statistically significant evidence that higher fuel diversity is associated with shorter average power outage durations.

While the observed differences suggest that higher diversity might relate to shorter outages, these differences could plausibly arise from random variation in the data.

### Discussion
Our decision to use Shannon entropy for fuel diversity captures the richness of a state's energy mix and aligns with established research. Testing both capacity and generation-based entropy helps us explore both infrastructure preparedness and actual operational diversity.

Although our results do not support a statistically significant relationship in this dataset, this does not imply that diversity has no effect. We purpose that limitations such as small sample size, unmeasured confounding variables (e.g., weather, utility regulations), and data quality may influence the results. 


---

# Framing a Prediction Problem

To get a closer look into the relationship between our features and outage duration, we have decided to make our task be about predicting the average outage duration per state per year through regression. By predicting this response variable, we aim to provide a model that can derive average outage duration data catered to each state to more accurately. We will propose a baseline model, starting with linear regression, as it is one of the simpler models, with percent RMSE, the percentage of RMSE over the mean of the average duration per state per year feature.   

Our features of interest include `climate_region`, `cause_cat`, `population`, `capacity_fuel_diversity_index`, and `generation_fuel_diversity_index`. These features are either fixed (e.g., geographic or categorical labels) or collected prior to the prediction year through public data sources, ensuring they would be available at the time of prediction. This respects the temporal constraint of not using information from after the event we aim to predict.   

---

# Baseline Model

First, we attempted Linear Regression using the covariates identified during the prediction problem formulation to predict the average outage duration per year per state. However, the results were not promising, as the RMSE was quite high. This is likely due to the nature of our test set. While the model learns from the training data with access to all covariates, it is important that its performance on the test set does not degrade significantly compared to training. If the model’s accuracy on the test set remains close to that on the training set, we consider it a good model because it demonstrates robustness to new, potentially independent data outside the training distribution.   
**Our Linear Regression model's test MSE is 8,455,566.45.**


We were suspicious of our findings, so we looked further into how further test our model's predictive power. We came across Dummy Regression where our model would try to fit a constant line and failed. So then, we can conclude that the model we have is not great at predicting. 

To improve performance, we turned to **Random Forest Regression**, which is more flexible than linear regression and better suited for capturing non-linear relationships in the data. In this model, we used the following features to predict the **average outage duration** for each state per year:

- `climate_region`
- `cause_cat`
- `population`
- `capacity_fuel_diversity_index`
- `generation_fuel_diversity_index`

The resulting **Mean Squared Error (MSE)** for the Random Forest model was **4,983,457.59**, which is a substantial improvement over the baseline linear regression model. This is possiblty due to Random Forest Regression is more suited for non linear relationships and has deeper and more diverse ways to reduce bias and variance because of the way Random Forests are constructed from the training set.


We can improve by choosing more variables to work with and let the RFG optimize based on that. We also want to be able to choose optimal hyperparameters in the form of tree depth and amount of trees to produce in order to find a balance between the bias and variance trade off. What we did here with RF was just not restricting its depth which can make it less optimal when getting prediction error.

| Metric                | Linear Regression | Random Forest       |
|-----------------------|-------------------|---------------------|
| R² Score              | 0.104             | 0.470               |
| Test MSE              | 8,455,566.45      | 4,996,551.57        |
| Test RMSE             | 111.28            | 85.54               |


### Feature Selection Rationale
+ Cause category and climate region (both are nominal as they do not have an apparent ordering) both are intuitively related to the duration of outages since they directly play into the reason for the outage occuring in the first place. So, it makes sense to include them to see if they have any contributions to predicting average duration. Both of them are going to be one hot encoded
+ For population (quantitative), it is important to tell how much electricity is being distributed to each household on average and interesting to see if this is indeed something closely related to average outage duration. 
+ capacity_fuel_diversity_index and generation_fuel_diversity_index are there to tell us information on the efficiency and supply of electricity. 

---

# Final Model

In order to narrow down which features to use, we will analyze multicollinearity to rule out columns that may not contribute much to our model given the features that are already in it.

<iframe
  src="assets/plots/covariance_heatmap.html"
  width="1000"
  height="1000"
  frameborder="0"
></iframe>


We sense that there is a strong multicollinearity pattern among the columns that are related to population, for example:

- `population`
- `pop_pct_urban`
- `pop_pct_uc`
- `popden_urban`
- `popden_uc`
- `popden_rural`
- `area_pct_urban`
- `area_pct_uc`


This is to be expected since they all, in one way or another tie back to the population of their respective states. This means that they are dependent columns and this introduces multicollinearity. That is to say, since we already have the population feature in our model, there isn't a need to include these other population-based features.

Other notable multicollinearity occurrences are between the Shannon Entropy columns against the columns in our original dataframe that correspond to geographical elements (i.e. not anomaly level, duration, avergae duration, or pc_realgsp_state). 

So, it might not be the best idea to use geographical elements along with the capacity and generation indeces.

Despite us showing you which columns have high correlation with one another, it is important to focus on the main objective with our model, **prediction**. And so, since multicollinearity does not affect our prediction power, only the statistical significance of the covariates, we need not worry about it for now. Although, it would be interesting to see.

We would like to one hot encode the state column to see if there are any states that have have a say in the average duration of outages. Based on our EDA, it would seem that this may be the case due to the sheer amount of outages happening in some states. For example CA, TX, and WA have yearly outage count of over 20.

### Proposition

Since **Random Forest Regression** shows promise in the baseline model, performing a lot better than Linear Regression, we will choose to use it here for our model refinement. Through GridSearch cross validation, we hope to fine tune its hyperparameters (depth for overfitting, number of trees for variance reduction)

Here's what tuning hyperparameters can do for us:

+ Each decision tree in the forest is trained on a random sample with replacement of the training data.
+ This sampling introduces diversity among the trees which is to say the trees are less dependent on each other and reduces bias.
+ The model then averages the predictions from all trees, which reduces variance of the ensemble (by the law of large numbers).

So we have decided to tune on the following hyperparameters:

| Hyperparameter      | What it Affects                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| `n_estimators`   (int)      | More trees generally reduce variance, but take longer                  |
| `max_depth`   (int)         | Limits complexity; larger values may overfit                           |
| `max_features`   (int)      | Controls the randomness of splits; smaller values = more diverse trees |
| `bootstrap`   (boolean)         | Enables bootstrapping (random sampling with replacement)               |

### Optimal model parameters
Our best model settings were as follows: we disabled bootstrapping, allowed trees to grow up to a maximum depth of 50, used 200 trees in the forest, and for each split, the model considered the square root of the total features. These choices together gave us the best performance.

### Improvement over baseline model 
Our final tuned Random Forest model shows a noticeable improvement compared to the baseline model. Key hyperparameters were adjusted as follows: the maximum tree depth was reduced from unlimited to 30 to prevent overfitting, the number of trees was decreased from 2000 to 500 for efficiency, the number of features considered at each split changed from using all features (auto) to the square root of features (sqrt), and bootstrapping was disabled.

These changes resulted in a reduction of the RMSE on the test set from 2235.30 to 2166.94, representing a 3.06% decrease in error. The percent RMSE similarly improved by approximately 2.62%, indicating better predictive accuracy and generalization of the tuned model over the baseline.


| **Metric**          | **Baseline RF Model** | **Tuned RF Model** | **Improvement** |
| ------------------- | --------------------- | ------------------ | --------------- |
| `max_depth`         | Default (None)        | 30                 | Tuned           |
| `n_estimators`      | 2000         | 500                | Increased       |
| `max_features`      | Default ('auto')      | 'sqrt'             | Changed         |
| `bootstrap`         | True                  | False              | Changed         |
| **RMSE (Test Set)** |   **2235.2967521114506** | **2166.9378956680084** | **↓ 3.0581557%**  |
| **Percent RMSE**    | **85.54457245365353%**  | **82.92848619641265%**   | **↓ 2.6161%**     |


---
# Fairness Analysis
To assess model fairness, we choose group the cause category by severe weather or no severe weather. Since our model focuses on average duration based on the predictive power of population, geographical, and energy generation/storage factors, we want to know if the model is actually performing well on severe weather or not. This is an interesting way to answer the question of "How good is the model in identifying natural causes and appropriate assign weights to those causes so that it is predicting average duration fairly?". We will see whether or not our model is robust under why the outage happened. 

it is fitting to test whether or not the model performs well on different groups of climate categories. Specifically, we want to see if the average outage duration is well modeled on severe weather and no severe weather in hopes to bring up awareness of risks that come with outages.

<iframe
  src="assets/plots/fairness.html"
  width="800"
  height="400"
  frameborder="0"
></iframe>

The permutation test yielded a p-value of approximately `0.003992`, which is well below the common significance threshold of 0.05. This indicates a statistically significant difference in the model’s predicted average outage durations between the severe weather and non-severe weather groups. In other words, the model’s performance varies depending on the cause category, suggesting it may not be equally accurate or fair across these groups.

This finding highlights that while the model captures general trends, it might under- or over-estimate outage durations associated with severe weather events compared to other causes. Addressing this disparity could be important to improve model fairness and ensure it reliably informs decisions related to natural cause outages.

---

# Contributors 

Jiaying Chen| [github](https://github.com/rcwoshimao) | [linkedin](https://www.linkedin.com/in/jiaying-chen01/)  
Duc Minh Hoang | [github](https://github.com/thekingofrice) | [linkedin](https://www.linkedin.com/in/duc-minh-hoang-711029296/)

