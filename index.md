---
layout: article
title: How Electric Generation Infrastructure and Fuel Diversity Influence Frid Resilience Across U.S. States
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
We investigate the relationship between generation capacity, fuel source diversity, and actual electricity generation (via EIA-860 and EIA-923) with outage frequency, duration, and impact (via DOE outage data). In this project, we: 
- Perform basic examination of outage data
- Explored the question "Do states with more diverse energy portfolios experiences fewer or shorter outages?" 
- ...And Created a prediction model utilizing features described above to predict the duration of an outage. 

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
#### Data cleaning 
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
We were originally going to using capacity per capita, but this didnt work because there are too many missing population values in state-year population pairs. After doing some research, we decided to engineer the features `capacity_fuel_diversity_index` and `generation_fuel_diversity_index`, by converting the info in our dataframe into **Shannon Entropy**. 

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
Clearly state your prediction problem and type (classification or regression). If you are building a classifier, make sure to state whether you are performing binary classification or multiclass classification. Report the response variable (i.e. the variable you are predicting) and why you chose it, the metric you are using to evaluate your model and why you chose it over other suitable metrics (e.g. accuracy vs. F1-score).

Note: Make sure to justify what information you would know at the “time of prediction” and to only train your model using those features. For instance, if we wanted to predict your final exam grade, we couldn’t use your Final Project grade, because the project is only due after the final exam! Feel free to ask questions if you’re not sure.

With the above features 

*Describe how you framed your problem as a prediction task, including defining inputs and outputs.*

---

# Baseline Model

First, we tried to use Linear Regression alone with all other covariates to predict the average duration of outages per year per state, but it was not fruitful as the RMSE was too high. This can be explained due to the nature of our testing set. As much as the model can learn from the training dataset with all the covariates in its hands, if it does not perform much worse than the training with the testing dataset, then we can consider it a good model. This is because it has shown some robustness against new, and possibly independent, data from outside the one it was trained from.

However, we were suspicious of our findings, and so we looked further into how further test our model's predictive power. We came across Dummy Regression where our model would try to fit a constant line and failed. So then, we can conclude that the model we have is not great at predicting.

To improve performance, we turned to **Random Forest Regression**, which is more flexible than linear regression and better suited for capturing non-linear relationships in the data. In this model, we used the following features to predict the **average outage duration** for each state per year:

- `climate_region`
- `cause_cat`
- `population`
- `capacity_fuel_diversity_index`
- `generation_fuel_diversity_index`

The resulting **Mean Squared Error (MSE)** for the Random Forest model was **4,983,457.59**, which is a substantial improvement over the baseline linear regression model.


We can improve by choosing more variables to work with and let the RFG optimize based on that. We also want to be able to choose optimal hyperparameters in the form of tree depth and amount of trees to produce in order to find a balance between the bias and variance trade off. What we did here with RF was just not restricting its depth which can make it less optimal when getting prediction error.

### Feature Selection Rationale
+ Cause category and climate region (both are nominal as they do not have an apparent ordering) both are intuitively related to the duration of outages since they directly play into the reason for the outage occuring in the first place. So, it makes sense to include them to see if they have any contributions to predicting average duration. Both of them are going to be one hot encoded
+ For population (quantitative), it is important to tell how much electricity is being distributed to each household on average and interesting to see if this is indeed something closely related to average outage duration. 
+ capacity_fuel_diversity_index and generation_fuel_diversity_index are there to tell us information on the efficiency and supply of electricity. 

| Metric                | Linear Regression | Random Forest       |
|-----------------------|-------------------|---------------------|
| R² Score              | 0.104             | 0.470               |
| Test MSE              | 8,455,566.45      | 4,996,551.57        |
| Test RMSE             | 111.28            | 85.54               |


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

