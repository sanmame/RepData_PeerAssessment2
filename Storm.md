---
title: "Severe weather events causing major human and property damage"
output: 
  html_document:
    keep_md: true
---



Synopsis: Storms and other severe weather events can cause both public health and economic problems for communities and municipalities. Many severe events can result in fatalities, injuries, and property damage, and preventing such outcomes to the extent possible is a key concern. This report determines the type of weather events that have caused the greater number of human damage and property damage across the United States in the last decade (2001-2011) by analyzing the U.S. National Oceanic and Athomspheric Administraions's storm database.


## 1. Data
The data comes from the U.S. National Oceanic and Atmospheric Administration's (NOAA) storm database. This database tracks characteristics of major storms and weather events in the United States, including when and where they occur, as well as estimates of any fatalities, injuries, and property damage. The database can be downloaded [here] (https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2).

The National [Weather Service Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf) contains information about how some of the variables are constructed.

The variables used in this report are the following:
- BGN_DATE: the beginning date of the weather event
- EVTYPE: the type of weather event
- FATALITIES: the number of fatalities caused by the event (direct, indirect and delayed)
- INJURIES: the number weather-related injuries (direct and indirect)
- PROPDMG and PROPDMGEXP: property damage estimates in actual dollar amounts

The events in the database start in the year 1950 and end in November 2011. In the earlier years of the database there are generally fewer events recorded, most likely due to a lack of good records. More recent years should be considered more complete.


## 2. Data Processing
First, the data is downloaded, unzipped and loaded.


```r
destfile="./repdata_data_StormData.csv" 
fileURL <- "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"   
if (!file.exists(destfile)) {
    download.file(fileURL ,destfile,method="auto")
    unzip("repdata_data_StormData.csv.bz2")
}
raw_data <- read.csv("repdata_data_StormData.csv")
```

Then, the relevant subset of variables is selected. A new BGN_YEAR variable is created by extracting the year from BGN_DATE. Data from 2001-2011 is filtered. EVTYPE values are normalized to upper case, the values including 'SUMMARY' are excluded and plural names are changed to singular.


```r
require(dplyr)
data <- select(raw_data, BGN_DATE, EVTYPE, FATALITIES, INJURIES, 
               PROPDMG, PROPDMGEXP)
data <- data %>% 
    mutate(BGN_YEAR=format(as.POSIXct(data$BGN_DATE, format = "%m/%d/%Y %H:%M:%S"), "%Y"))  %>% 
    filter(BGN_YEAR > 2000)
data$EVTYPE <- toupper(as.character(data$EVTYPE))
data <- data %>% filter(!grepl("SUMMARY.*",EVTYPE))
data$EVTYPE <- gsub("(ES|S)$","", data$EVTYPE)
data$EVTYPE <- gsub("^[[:space:]]+","", data$EVTYPE)
data$EVTYPE <- as.factor(data$EVTYPE)
```


### 2.1 Types of events most harmful with respect to population health
In this section, we will analyze the types of weather events that are more harmful across the United States with respect to population health. The damage to population health is divided into injuries and fatalities.

First, a data frame with the sum of fatalities and injuries per year and event type is created. 


```r
hu_data <- data %>% select(BGN_YEAR, EVTYPE, FATALITIES, INJURIES)
hu_data_sum <- hu_data %>% group_by(BGN_YEAR,EVTYPE) %>% dplyr::summarize(FATTOTAL=as.numeric(sum(FATALITIES)), INJTOTAL=as.numeric(sum(INJURIES)))
hu_data_sum <- as.data.frame(hu_data_sum)
```

Second, we plot the number of fatalities by event type across time. Observations with zero fatalities are filtered. The dataset is restricted to event types with a number of fatalities above the mean of fatalities by type and year to make the plot more manageable.


```r
hu_data_sum_fatalities <- hu_data_sum %>% filter(!FATTOTAL == 0)
hu_data_sum_fatalities <- hu_data_sum_fatalities %>% filter(FATTOTAL > mean(FATTOTAL))
```


```r
require(ggplot2)
qplot(BGN_YEAR, FATTOTAL, data=hu_data_sum_fatalities, color=EVTYPE, group=EVTYPE) + geom_line() + labs(y="Number of fatalities", x="year", title="Total number of fatalities by event type")
```

![](Storm_files/figure-html/fatalities_per_event_type-1.png)<!-- -->

Third, we plot the number of injuries by event type across time. A data frame with the sum of injuries per year and event type is created with the same filters as above.


```r
hu_data_sum_injuries <- hu_data_sum %>% filter(!INJTOTAL == 0)
hu_data_sum_injuries <- hu_data_sum_injuries %>% filter(INJTOTAL > mean(INJTOTAL))
qplot(BGN_YEAR, INJTOTAL, data=hu_data_sum_injuries, color=EVTYPE, group=EVTYPE) + geom_line() + labs(y="Number of injuries", x="year", title="Total number of injuries by event type")
```

![](Storm_files/figure-html/injuries_per_event_type-1.png)<!-- -->

The event type that caused the maximum number of injuries and fatalities is calculated next.


```r
fatalities_sum <- hu_data_sum %>% group_by(EVTYPE) %>% dplyr::summarize(FATTOTALTYPE=as.numeric(sum(FATTOTAL)))
fatalities_sum <- as.data.frame(fatalities_sum)
fatalities_max <- fatalities_sum[which.max(fatalities_sum$FATTOTALTYPE),]$EVTYPE
fatalities_max_amount <- fatalities_sum[which.max(fatalities_sum$FATTOTALTYPE),]$FATTOTALTYPE

injuries_sum <- hu_data_sum %>% group_by(EVTYPE) %>% dplyr::summarize(INJTOTALTYPE=as.numeric(sum(INJTOTAL)))
injuries_sum <- as.data.frame(injuries_sum)
injuries_max <- injuries_sum[which.max(injuries_sum$INJTOTALTYPE),]$EVTYPE
injuries_max_amount <- injuries_sum[which.max(injuries_sum$INJTOTALTYPE),]$INJTOTALTYPE
```


### 2.2 Types of events with greatest economic consequences
In this section, we determine the types of events that have the greatest economic consequencies across the United States. "Economic consequences" is understood as property damage and crop damag

First, the values for property damage (PROPDMG) are normalized by changing them to their corresponding power of 10. In PROPDMGEXP, “K” stands for thousands, “M”, for millions, and “B”, for billions. The numbers from one to ten represent the power of ten. The symbols "-", "+" and "?" refers to less than, greater than and low certainty and are ignored (10^0). The normalized values are stored in the new variables PROPDMGTOTAL.


```r
require(plyr)
ec_data <- data %>% select(BGN_YEAR, EVTYPE, PROPDMG, PROPDMGEXP)
ec_data$PROPDMGEXP <- as.character(ec_data$PROPDMGEXP)
ec_data$PROPDMGEXP <- revalue(ec_data$PROPDMGEXP, 
                           c("H" = as.numeric(2), "h" = as.numeric(2),
                             "K" = as.numeric(3), "k" = as.numeric(3),
                             "M"= as.numeric(6), "m" = as.numeric(6),
                             "B"= as.numeric(9), "b" = as.numeric(9),
                             "+" = as.numeric(0), "-" = as.numeric(0), "?" = as.numeric(0)))
ec_data$PROPDMGEXP <- as.numeric(ec_data$PROPDMGEXP)
ec_data <- ec_data %>% 
    mutate(PROPDMGTOTAL=PROPDMG * 10^PROPDMGEXP)
```

Second, a data frame with the sum of property damage in dollars per year and event type is created. Observations with zero or NA are filtered. The dataset is restricted to event types with a number of fatalities above the mean of fatalities by type and year to make the plot more manageable. 


```r
ec_data_sum <- ec_data %>% group_by(BGN_YEAR,EVTYPE) %>%
  dplyr::summarize(PROPDMGEVTYPE=as.numeric(sum(PROPDMGTOTAL, na.rm=TRUE)))
ec_data_sum <- as.data.frame(ec_data_sum)
ec_data_sum_prop <- ec_data_sum %>% filter(!PROPDMGEVTYPE == 0)
ec_data_sum_prop <- ec_data_sum_prop %>% filter(PROPDMGEVTYPE > mean(PROPDMGEVTYPE))
```

Third, the plot is drawn.

```r
require(ggplot2)
qplot(BGN_YEAR, PROPDMGEVTYPE, data=ec_data_sum_prop, color=EVTYPE, group=EVTYPE) + geom_line() + labs(y="Property damage (in dollars)", x="year", title="Property damage by event type")
```

![](Storm_files/figure-html/property_damage_per_event_type-1.png)<!-- -->

The event type that caused the maximum amount of property damage is calculated next.


```r
prop_sum <- ec_data_sum %>% group_by(EVTYPE) %>% 
  dplyr::summarize(PROPDMGEVTYPE=as.numeric(sum(PROPDMGEVTYPE, na.rm = TRUE)))
prop_sum <- as.data.frame(prop_sum)
prop_max <- prop_sum[which.max(prop_sum$PROPDMGEVTYPE),]$EVTYPE
prop_max_amount <- prop_sum[which.max(prop_sum$PROPDMGEVTYPE),]$PROPDMGEVTYPE
```


## 3. Results

Regarding the types of events most harmful with respect to population health, the event type that caused the maximum number of fatalities in the last 10 years is TORNADO (1152 deaths) and the one that caused the maximum number of injuries is TORNADO (1.4331\times 10^{4} people injured).

As for the type of event that caused the greater property damage, it is FLOOD, which damaged property for 1.3397242\times 10^{11} dollars.

