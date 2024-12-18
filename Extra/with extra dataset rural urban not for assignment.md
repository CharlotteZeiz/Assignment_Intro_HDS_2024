---
title: "Assignment Intro HDS - Infant Death Scotland"
author: "Sophie Charlotte Zeiz"
date: "2024-11-18"
output: pdf_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

# Infant mortality in Scotland\

For this assignment, I wanted to look into the data from the **Scottish Public Health Observatory** regarding changes and geographical patterns in infant mortality. "The infant mortality rate is defined as the number of children who die before reaching their first birthday in a given year, expressed per 1 000 live births." ([OECD/World Health Organization (2018)](https://doi.org/10.1787/health_glance_ap-2018-8-en)). Ending preventable deaths of newborns and children under the age of five is one of the World Health Organisations sustainable development goals [1].\ 
My target audience are health officers working in the NHS or the councils, as they could initiate further research or data acquisition.
The dataset used for this assignment was extracted from the [ScotPHO Online Profiles Tool](https://scotland.shinyapps.io/ScotPHO_profiles_tool/) and is freely available for the public. 
For additional analysis on geographical patterns, two datasets from  **Public Health Scotland** were added: 
1. the Urban-Rural-Classification was used to display differences in mortality rate between urban an rural areas in six categories (https://www.opendata.nhs.scot/dataset/urban-rural-classification)
2. to link both datasets, a codes and labels dataset from (https://opendata.scot/datasets/public+health+scotland-geography+codes+and+labels/)\

##Preparation
###Load packages
```{r load packages, message=FALSE} 
#loading the packages i used most during the course
library(readr)
library(dplyr)
library(here)
library(viridisLite)
```

###Read data from ScotPHO
```{r load data scotpho, message=FALSE}
#Read in data from ScotPHO

library(readr)
infant_death_all_geo <- read_csv("ScotPHO_infantdeath_all_update.csv")
```

###Inspect data
```{r inspect, message=FALSE}
glimpse(infant_death_all_geo) #to see what data dataset contains
any(is.na(infant_death_all_geo)) #to see if there are missing values (FALSE = no NA)
```

##Data cleaning
```{r Clean data, chose variables, message=FALSE}
#Selecting values i need for my first visualisation

mortality_allgeo <- infant_death_all_geo %>% 
  select(area_code, 
         area_type, 
         area_name, 
         year,
         measure) %>% 
  rename(mortality_rate = `measure`) 
```

## Infant mortality rate in Scotland between 2004-2019
###Graph 1
This line graph shows the evolution of the infant mortality rate. Between 2004 and 2018 we can observe a reduction in infant mortality, in 2019 the rate got slightly higher again.
```{r visualize infant mortality rate 2004-19}
#visualize graph 
library(ggplot2)
mortality_allgeo %>% 
  select(year, mortality_rate, area_type) %>% 
  filter(area_type == "Scotland") %>%
  ggplot() +
  geom_line(aes(x = year, y = mortality_rate), linewidth = 1)+
  labs(x = "Year",
       y = "Infant mortality per 1000 live births") +
  theme_bw()+
   scale_y_continuous(limits = c(0, 5)) #adjusting the scale to include zero -> more realistic visualisation
```
###Graph 2
To display the reduction of infant mortality rate within Scotland, i chose to create boxplots using the data from the 33 council areas. The boxplots show


```{r select and prepare, boxplot, skewed or normal data -> SD or IQR, message=FALSE}
#data for boxplot 
mortality_scot_by_council <- mortality_allgeo %>% 
  select(`area_type`, 
         `area_name`, 
         `year`,
         `mortality_rate`) %>%
  filter(`area_type` == "Council area",
         `year` %in% c("2004", "2007", "2010", "2013", "2016", "2019")) #every 3 years

mortality_scot_by_council$year <- as.factor(mortality_scot_by_council$year) #to help separate boxplots per year in one graph


#check if data is normal distrusted or skewed
#Histogram
mortality_scot_by_council %>% 
  ggplot(aes(x = mortality_rate))+
  geom_histogram()

#Q-Q-plot
mortality_scot_by_council %>% 
  ggplot(aes(sample = mortality_rate))+
           geom_qq()+
  geom_qq_line()

#data is skewed --> IQR instead of sd

#IQR values per year
IQR_data <- mortality_scot_by_council %>%
  group_by(year) %>%
  summarise(
    Q1 = quantile(mortality_rate, 0.25, na.rm = TRUE),
    Q3 = quantile(mortality_rate, 0.75, na.rm = TRUE),
    IQR = Q3 - Q1)
```


```{r select and prepare, boxplot, skewed or normal data -> SD or IQR, message=FALSE}
#boxplot
library(ggplot2)

p2 <- mortality_scot_by_council %>% 
  ggplot(aes(x = factor(year),
             y = mortality_rate,
             colour = year)) +
  geom_boxplot() +
  geom_jitter(alpha = 0.6) +
  theme(legend.position = "none")+
  labs(x = "Year", 
       y = "Infant mortality rate", 
       title = "Infant mortality rate in Scotland by council areas (2004-2019)")+
  scale_y_continuous(limits = c(-0.5, 12))
 
p3 <- IQR_data %>% 
  ggplot()+
  geom_point(aes(x = year, y = IQR))+
   scale_y_continuous(limits = c(0.5, 2))

library(patchwork)
p2/p3 +
  plot_layout(heights = c(5, 2))
```




##Linking Urban-Rural classification
```{r load rural urban classification data, message=FALSE}
library(readr)
rural_class_complete <- read_csv("scottish-government-urban-rural-classification-2020-data-zone-2011-lookup.csv")
View(rural_class_complete)
```
```{r clean urban rural to 6 categories, message=FALSE}
rural_6fold <- rural_class_complete %>% 
  select(`DataZone`, `UR6FOLD`) %>% 
  rename(area_code = `DataZone`,
         classification = `UR6FOLD`
         )
```

```{r combine urban-rural and area dataset, message=FALSE}
#read area dataset
library(readr)
area_to_hsc <- read_csv("dz2011_codes_and_labels_21042020.csv")


# connecting area code and hsc for 6 fold
CA_rural6fold <- rural_6fold %>%
  inner_join(`area_to_hsc`, by = c("area_code" = "DataZone")) %>% 
  select(area_code, classification, CA, HSCP, HB)
```
```{r get mode with help of chat gtp, message=FALSE}
library(dplyr)
get_mode <- function(x) {
  uniq_x <- unique(x)
  uniq_x[which.max(tabulate(match(x, uniq_x)))]}

# Combine council area, mortality rates and rural-urban classification

combined_mortalityallgeo_rural6fold <- mortality_allgeo %>%
  inner_join(`CA_rural6fold`, by = c("area_code" = "CA")) %>% 
  select(area_type, area_name, year, mortality_rate, classification)

#summarise mortality and classification per area_name and year 
AVG_6_mort_class_percouncil_peryear <- combined_mortalityallgeo_rural6fold %>%
  group_by(area_name, year) %>% 
  summarise(
    avg_mortality_rate = mean(mortality_rate, na.rm = TRUE),
    mode_classification = get_mode(classification) #mode for whole numbers of original categories
  )

#create classification categories via mode according to scot public health
 group_classification <- AVG_6_mort_class_percouncil_peryear %>% 
  mutate(mode_classification = case_when(
    mode_classification == 1 ~ "Large Urban Areas",
    mode_classification == 2 ~ "Other Urban Areas",
    mode_classification == 3 ~ "Accessible Small Towns",
    mode_classification == 4 ~ "Remote Small Towns",
    mode_classification == 5 ~ "Accessible Rural",
    mode_classification == 6 ~ "Remote Rural",
    TRUE ~ "Unknown"
  ))

# average mortality rate per classification
avg_6class_mort_year <- AVG_6_mort_class_percouncil_peryear %>%
  select(`year`,`avg_mortality_rate`, `mode_classification`) %>%
  group_by(mode_classification, year) %>%
  summarise(AVG_mortality_rate = mean(avg_mortality_rate, na.rm = TRUE))

#r Perform Kruskal-Wallis test for 3 and 6 fold, include=FALSE}
kruskal_test_result <- kruskal.test(AVG_mortality_rate ~ mode_classification, data = avg_6class_mort_year)
kruskal_test_result
# -->continue with the 6-fold dataset
```

```{r average mort rate per text classification, message=FALSE}
avg_class_mort_year <- group_classification %>%
  select(`year`,`avg_mortality_rate`, `mode_classification`) %>%
  group_by(mode_classification, year) %>%
  summarise(avg_mortalityrate = mean(avg_mortality_rate, na.rm = TRUE))
```
```{r, message=FALSE}
library(ggplot2)

#to keep order in the legend
avg_class_mort_year$mode_classification <- factor(avg_class_mort_year$mode_classification, levels = c("Large Urban Areas", "Other Urban Areas", "Accessible Small Towns", "Accessible Rural", "Remote Rural"))

avg_class_mort_year %>%
  select(`year`,`avg_mortalityrate`, `mode_classification`) %>%
  group_by(`mode_classification`) %>% 
  ggplot(alpha= 1) +
  geom_line(aes(x= `year`, y= `avg_mortalityrate`, colour =`mode_classification`),size=1) +
              labs(x = "Year", y = "Infant mortality rate", title = "Infant mortality rate in Scotland by urban-rural classification (2004-2019)")+
  theme(legend.position ="right")
```
 
  




[1]: https://www.who.int/data/gho/data/themes/topics/sdg-target-3_2-newborn-and-child-mortality)