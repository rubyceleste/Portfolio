# 🗺️ Ruby's Portfolio

I am an environmental data scientist with a Master of Professional Science in Marine Conservation from the University of Miami. My research focuses on dendrochronology, forest carbon dynamics, and climate impacts on biomass growth. I specialize in transforming complex ecological datasets into meaningful visualizations and predictive models using R.

My experience includes data wrangling, statistical modeling, Bayesian analysis, time-series forecasting, and producing publication-quality images that communicate scientific findings to both technical and non-technical audiences.

## 📚 Table of Contents 

- [Skills](#skills)
- [Projects](#projects)
    - [Climate change and biomass accumulation](#biomass)
    - [Estimating Holocene albedo using fossil-pollen data](#holocene-albedo)

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

<a id="biomass"></a>
## 🌳 The effects of climate change on aboveground biomass accumulation

This project began as a an internship requirement for my MPS degree, the full report can be found [here](https://scholarship.miami.edu/esploro/outputs/report/Forest-carbon-accumulation-patterns-and-drivers/991032728735502976?institution=01UOML_INST) This work is ongoing. 
Tree-ring data and various climate variables across the Northeastern United States were used to forecast the effects of climate change on aboveground biomass accumulation. 
The goal of this research is to determine the predictability of biomass accumulation and explore what may be driving growth and how accurate our predictions might be. 
<img width="992" height="744" alt="image" src="https://github.com/user-attachments/assets/fd7d7162-4da9-4c02-9d66-2913b5574d43" />

#

### Summarizing tree-ring data
Tree-ring data was summarized in a STAN statistical model to estimate aboveground biomass for each individual tree. 1 chain was run with 5000 iterations, only the last 250 iterations were considered for biomass estimates. 
This data was provided to me from previous research.

All iterations were summarized for each species found at each site, where we have total aboveground biomass and increment over time. 


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

Climate data was pulled from the PRISM climate database: monthly values of minimum, maximum, mean temperature, minimum, maximum, VPD, and precipitation. Which was then  summarized into seasonal values. 

```R
#summarizing precipitation into seasonal data
clim_seasons = clim_agbi_taxon %>% 
  group_by(year, site, taxon) %>% 
  mutate(PPT_winter = sum(dplyr::pick('PPT_12', 'PPT_01', 'PPT_02')),
         PPT_spring = sum(dplyr::pick('PPT_03', 'PPT_04', 'PPT_05')),
         PPT_summer = sum(dplyr::pick('PPT_06', 'PPT_07', 'PPT_08')))
```


### Correlation between aboveground biomass increment and climate data

Prior to model development, correlation between aboveground biomass increment and climate data was run. 

<img width="716" height="476" alt="image" src="https://github.com/user-attachments/assets/1fb24d7e-fc62-4313-9c6b-bc2defd536b5" />



### ARIMA model 

Two models were then created: a taxon model and a site model. Twenty predictors were input into the model which is the seasonal climate data of each variable. The ARIMA model was chosen for its autoregressive nature as past biomass increment growth is considered to influence future growth.  

```R
#uses seasonal climate data 
#making a time series 
# Fit ARIMA models by site and taxon
models <- clim_agbi_in %>%
  dplyr::group_by(site, taxon) %>%
  dplyr::do({
    df <- .
    # 
    # Drop rows with NA in response or predictors
    if (any(is.na(df$AGBI.mid)) || any(is.na(df[predictor_names]))) {
      dplyr::tibble(mod = list(NULL), note = "Missing data")
    }
    
    # Response variable as time series
    agbi_ts <- ts(df$AGBI.mid, start = min(df$year), frequency = 1)

    # External regressors
    xreg <- as.matrix(df %>% dplyr::select(dplyr::all_of(predictor_names)))
    
    # Fit ARIMA model
    model <- tryCatch({
      forecast::Arima(
        y = agbi_ts,
        order = c(1, 0, 0),  # AR(1)
        xreg = xreg,
        method = "ML"
      )
    }, error = function(e) {
      warning("ARIMA failed for site=", df$site[1], ", taxon=", df$taxon[1], ": ", e$message)
      return(NULL)
    })
    
    tibble(mod = list(model), note = if (is.null(model)) "Model failed" else NA)
  })
```

### Forecasting 

Using the ARIMA model, biomass increment can be forecasted and the model can be used to predict future growth based on the changing climate. 

```R
#forecasting
model_forecasts <- models %>%
  filter(!is.null(mod[[1]])) %>%
  mutate(
    forecast = pmap(list(mod, site, taxon), function(model_obj, site_val, taxon_val) {
      if (is.null(model_obj)) return(NA)
      
      # Get future predictors for the same group
      future_data <- clim_agbi_out %>%
        filter(site == site_val, taxon == taxon_val) %>%
        arrange(year)
      
      if (nrow(future_data) == 0) return(NA)
      
      future_xreg <-future_data %>%
        dplyr::ungroup() %>% 
        dplyr::select(dplyr::all_of(predictor_names)) %>% 
        as.matrix()
      
      # Forecast
      tryCatch({
        forecast::forecast(model_obj, xreg = future_xreg)
      }, error = function(e) {
        warning(paste("Forecast failed for", site_val, taxon_val, ":", e$message))
        return(NA)
      })
    })
  )
```

The model was developed using five fewer years of our present data. Using the created model those last five years were forecasted and compared to our present data to see how well our model could predict biomass increment. 

<img width="715" height="476" alt="image" src="https://github.com/user-attachments/assets/b95944a7-3043-431e-bc00-6380caf2b750" />

### Model Validation
Pairwise correlations of the residuals were taken for each species at each site.

This figure shows the pairwise correlations at the site Goose. Species where there are high positive correlations were investigated, as we believe there might be another factor that is not considered in our model that is impacting growth. 

<img width="435" height="430" alt="image" src="https://github.com/user-attachments/assets/6ed73f2b-71fb-4320-869e-f1c5c25ccacc" />


### Summary

This research is ongoing, however we can see that a majority of our forecasted values fall within the confidence interval for this site. Our current next steps are to consider how we can take disturbance events into account and quantify the predictability of biomass accumulation.  

The full scripts and data for this research can be found [here.](https://github.com/PalEON-Project/RW-2-BIO/tree/master/reboot)


<a id="holocene-albedo"></a>
## ☀️ Holocene albedo change estimated using fossil pollen data

This research uses fossil pollen data from (period) to estimate Holocene albedo change. Present (200X-XXXX) albedo data was pulled from .... and used to create a calibration model. 


UNDER CONSTRUCTION :)



