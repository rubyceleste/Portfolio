# 🗺️ Ruby's Portfolio

I am an environmental data scientist with a Master of Professional Science in Marine Conservation from the University of Miami. My research focuses on dendrochronology, forest carbon dynamics, and climate impacts on biomass growth. I specialize in transforming complex ecological datasets into meaningful visualizations and predictive models using R.

My experience includes data wrangling, statistical modeling, Bayesian analysis, time-series forecasting, and producing publication-quality images that communicate scientific findings to both technical and non-technical audiences.

## 📚 Table of Contents 

- [Skills](#Skills)
- [Projects](#Projects)



## Skills

### Programming
- R
- Stan

### Packages
- ggplot2
- dplyr
- tidyr
- forecast
- mgcv
- sf
- raster

### Methods
- Bayesian hierarchical modeling
- ARIMA forecasting
- Regression
- Time-Series analysis
- Data visualization
- Statistical interference

# Projects 

## 🌳The effects of climate change on aboveground biomass accumulation

This research using tree-ring data and various climate variables to forecast the effects of climate change on aboveground biomass accumulation. 
The climate variables included are: minimum, mean and maximum temperature, precipitation, and maximum vapor pressure deficit.


### Summarizing tree-ring data
Tree-ring data was summarized in a STAN statistical model to estimate aboveground biomass for each individual tree. 1 chain was run with 5000 iterations, only the last 250 iterations were considered for biomass estimates. 
This data was provided to me from previous research.

All iterations were summarized for each species found at each site. Where we have total aboveground biomass and increment over time. 


``` R
all_taxon_site_summary = all_taxon_site_by_iter %>%
  group_by(year, taxon, model, site) %>% 
  dplyr::summarize(AGB.mean = mean(AGB.iter.mean, na.rm = TRUE),
                   AGB.sd = sd(AGB.iter.mean, na.rm=TRUE),
                   AGB.lo = quantile(AGB.iter.mean, c(0.025), na.rm=TRUE),
                   AGB.hi = quantile(AGB.iter.mean, c(0.975), na.rm=TRUE), 
                   AGBI.mean = mean(AGBI.iter.mean, na.rm = TRUE),
                   AGBI.sd = sd(AGBI.iter.mean, na.rm = TRUE),
                   AGBI.lo = quantile(AGBI.iter.mean, c(0.025), na.rm=TRUE),
                   AGBI.hi = quantile(AGBI.iter.mean, c(0.975), na.rm=TRUE), 
                   .groups='keep')
head(all_taxon_site_summary)

saveRDS(all_taxon_site_summary, "reboot/AGBI_taxon_data.RDS")

```

This figure shows biomass increment over time at each of our sites. 

<img width="972" height="549" alt="site biomass increment over time" src="https://github.com/user-attachments/assets/ac0b3006-9064-4636-873e-bc6dbacb6708" />

### Summarizing climate data

Climate data was pulled from the PRISM climate database of monthly values of minimum, maximum, mean temperature, minimum, maximum, VPD, and precipitation. Climate data was summarized into seasonal values. 

```R
#summarizing precipitation into seasonal data
clim_seasons = clim_agbi_taxon %>% 
  group_by(year, site, taxon) %>% 
  mutate(PPT_winter = sum(dplyr::pick('PPT_12', 'PPT_01', 'PPT_02')),
         PPT_spring = sum(dplyr::pick('PPT_03', 'PPT_04', 'PPT_05')),
         PPT_summer = sum(dplyr::pick('PPT_06', 'PPT_07', 'PPT_08')))
```

### Correlation between aboveground biomass increment and climate data

<img width="716" height="476" alt="image" src="https://github.com/user-attachments/assets/1fb24d7e-fc62-4313-9c6b-bc2defd536b5" />


### ARIMA model 

Two models were created: a taxon model and a site model. 
20 variables input into the model
Each climate variable had seasonal data. 

19?


description 




