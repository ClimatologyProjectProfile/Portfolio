---
permalink: /projects/crimetology/
title: "Crimetology"
header:
  teaser: /assets/images/crimetology_image_small.png
layout: home_norecent
author_profile: true
use_math: true
mathjax: true
---
<!-- Some Bages -->
![Python](https://img.shields.io/badge/Python-3.12-blue.svg)
![Data Science](https://img.shields.io/badge/Domain-Data%20Science-green.svg)



# Crimetology: How Weather Shifts Spatial Crime Patterns

A data science project investigating how weather influences crime occurrence across Leisure versus Retail areas.

Reproduce This Project Yourself:

| Resource | Description | Link |
| --- | --- | --- |
| **Crime Data** | Raw UK Police street-level data archive | [data.police.uk](https://data.police.uk/data/archive/) |
| **Weather Data** | HadUK-Grid 1km gridded climate dataset | [CEDA Archive](https://catalogue.ceda.ac.uk/uuid/789b3065d74a4c948ab05d33556c86d0/)|
| **Code** | Complete Python ETL, analysis, & modeling scripts | [GitHub Crimetology Repo](https://github.com/ClimatologyProjectProfile/crimetology/tree/main) |


## Table of Contents
* [Executive Summary](#executive-summary)
* [Data Pipeline & Engineering](#1-the-data-pipeline--engineering-challenge)
* [Spatial Clustering & Proxy Context Labeling](#2-spatial-clustering--proxy-context-labeling)
* [Statistical Inference & Findings](#3-statistical-inference--findings)
* [Final Remarks and Limitations](#4-final-remarks-and-limitations)

## Executive Summary

Combining weather observations and open policing data enables a powerful dataset to test if temperature and relative humidity changes the likelihood of where crime occurs. 

By joining 1.26 million police records with gridded HadUK climate data, this project evaluates if weather drives crime towards outdoor leisure spaces versus commercial retail zones.

> **Key Finding:** Temperature and relative humidity exhibit a joint effect on the odds of where crime happens.    

__When considered together (Multivariate Model)__:  
Every 1°C increase in maximum temperature increases the odds of crime in a Leisure environment by 3.2%.     

Every 1% increase in relative humidity increases the odds of crime in a Leisure environment by 2.1%.  

When combines this increases the odds of crime events in Leisure zones by **5.37%**

![Fig1: Understand Odds]({{ '/assets/images.png' | relative_url }})
*Figure 1:*


__When considered separately (Univariate Models)__:  
Every 1°C increase in maximum temperature increases the odds of crime in a Leisure environment by 1.4%.  

Relative humidity changes have no statistically significant impact on odds.  

> This is a classic example of the *Suppressor Effect*. Temperature and humidity affect human behaviour together and push events into leisure zones. Therefore, we need two predictor variables to capture the odds properly. 





## 1. The Data Pipeline & Engineering Challenge

To establish a transparent, reproducible workflow without enterprise database costs, an end-to-end ETL pipeline was engineered using Python, DuckDB, and SciPy


* **Data Ingestion:** Automated Python scripts extract, standardize, and log monthly UK Police street-level data into a local DuckDB instance.
* **Handling Missing Identifiers:** Synthetic IDs were generated for ~16% of records with missing reference numbers to avoid sample-selection bias. Tracking over time revealed an overall improvement in upstream archive quality.
* **Spatial Matching:** Implemented a `scipy.spatial.cKDTree` nearest-neighbor lookup to pair each crime location with 1km-gridded CEDA HadUK weather observations.

![Fig2: Synthetic ID Quality Control](https://github.com/ClimatologyProjectProfile/Portfolio/blob/master/assets/images/fig2_synID.png)  
*Figure 2: Time-series monitoring of synthetic crime IDs (2015–2025), showing a downward quality-control trend and seasonal update spikes.*




## 2. Spatial Clustering & Proxy Context Labeling

Open police records anonymize exact crime coordinates and lack explicit land-use labels (e.g., "nightlife district" vs. "shopping centre"). To solve this, **DBSCAN density clustering** was applied to group 1.26M crime records into **9,200+ spatial clistures** across Norfolk and Suffolk.

![Fig3: Norfolk and Suffolk Boundary Dataset](https://github.com/ClimatologyProjectProfile/Portfolio/blob/master/assets/images/fig3_boundary_data.png)  
*Figure 3: Constrained study area consisting of ~1.26 million spatially filtered crime records across Norfolk and Suffolk.*

### Engineering Operational Labels
Clusters were categorized into **Leisure**, **Retail**, or **Other** contexts based on their internal crime-type composition:
* **Retail Context:** High proportion of shoplifting, commercial theft, and burglary.
* **Leisure Context:** High proportion of public order offences, anti-social behavior, and violence.

![Fig4: Crime Type Skew Histograms](https://github.com/ClimatologyProjectProfile/Portfolio/blob/master/assets/images/fig4_histograms.png)  
*Figure 4: Distribution of Retail, Leisure, and Other crime-type prevalence across clusters, supporting quantile-based context classification.*

Histograms revealed that Retail-associated crime distributions were heavily right-skewed, justifying quantile-based threshold over arbitrary, say 50%, cut-offs. Clusters in the top 10% of each crime type occcuring (leisure or retail) where given a label. Every other cluster was treated as 'mixed' and not labelled.  


![Fig5: Clusters with lables](https://github.com/ClimatologyProjectProfile/Portfolio/blob/master/assets/images/lablelled_clusters.png)  
*Figure 5: Clusters given a context label. Yellow = Retail label, Blue = Leisure Label.*



## 3. Statistical Inference & Findings

A Binary Logistic Regression model evaluated whether maximum temperature and relative humidity predicted whether a crime occurred in a Leisure versus Retail cluster.  


Using the exact coefficients from the Logit model we can plot how weather translates into predicted probabilities of crime occurring in a Leisure context.


![Figure6: Bivariate Logistic Regression Curves](https://github.com/ClimatologyProjectProfile/Portfolio/blob/master/assets/images/probability_leisure_context.png)

*Figure 6: The joint atmospheric effect, this chart demonstrates how temperature and relative humidity work together. High humidity shifts the entire curve upward, increasing the odds of leisure-context crimes across all temperature ranges.*

### How did we get to the above figure? 

The fundamental equation for a bivariate logistic regression model is written in terms of log-odds (the logit function):

$$\text{logit}(P) = \ln\left(\frac{P}{1 - P}\right) = \beta_0 + \beta_1 X_1 + \beta_2 X_2$$

**Where:**
* $P$ = Probability of a crime occurring in a **Leisure** context ($1$) versus a **Retail** context ($0$).
* $\frac{P}{1 - P}$ = The odds of the event occurring.
* $X_1$ = Mean maximum daily temperature (`tasmax`).
* $X_2$ = Relative humidity (`hurs`).


Substituting the predictor variable constants returned by [Statsmodels Logit](https://www.statsmodels.org/stable/generated/statsmodels.discrete.discrete_model.Logit.html#statsmodels.discrete.discrete_model.Logit) into the equation

* **Intercept ($\beta_0$):** $-3.5269$
* **Temperature Coefficient ($\beta_1$):** $0.0315$
* **Humidity Coefficient ($\beta_2$):** $0.0205$


we get our bivariate model as

$$\ln\left(\frac{P}{1 - P}\right) = -3.5269 + 0.0315(\text{tasmax}) + 0.0205(\text{hurs}).$$

Rearage this gives you

$$P(\text{Leisure}) = \frac{1}{1 + e^{-(-3.5269 + 0.0315 \cdot \text{tasmax} + 0.0205 \cdot \text{hurs})}}$$

which you can use with humidity set to 50% and 80% to plot the red and blue lines respectively. 



## 4. Final Remarks and Limitations

While this project demonstrates the feasibility of combining meteorological observations with policing data, it is only an exploratory tool. The findings highlight how dynamic environmental factors, such as thermal comfort could change crime patterns. This project corroborates wider literature on this topic. 

### Key Limitations
* **Spatial Anonymization:** Open police crime records are published with spatial anonymization applied to location coordinates, the recorded location could be up to 20km away from the actual location.
* **Temporal Resolution:** Monthly aggregated weather metrics are use to align with the monthly reporting resolution of open crime data, masking short-term daily or hourly temperature spikes.
* **Proxy Context Labels:** Land-use contexts ('Leisure' vs. 'Retail') were engineered using localized crime-type compositions as proxy measures. These could be wrong.
* **External Disruptions:** This data includes covid-19 records (2020–2021). Behavior would have shifted in this time frame so this may have impacted the data.




