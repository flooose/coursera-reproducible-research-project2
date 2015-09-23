---
title: "Storm Analysis"
output:
  html_document:
    keep_md: true
---

# NOAA Severe Weather Analysis

## Synopsis

The following is an analysis of weather data from National Weather
Service. It attempts to answer two questions, one, which types of
events are most harmful with respect to population health, and two,
which events have the greatest economic consequences.

## Data Processing

Setup:


```r
library(dplyr)
```

Loading the data:


```r
weatherData <- read.csv('repdata-data-StormData.csv.bz2')
```

### Question 1: Impact on human population.

First we take the subset of `FATALITIES` and `INJURIES` from the data
and create a new column `combinedDeathsAndInjuries`:


```r
populationImpact <- select(weatherData, FATALITIES, INJURIES)
populationImpact$combinedDeathsAndInjuries <- populationImpact$FATALITIES + populationImpact$INJURIES
```

Now we can aggregate the by event type:


```r
deathsAndInjuries <- aggregate(populationImpact, by=list(eventType = weatherData$EVTYPE), sum, na.rm = T)
deathsAndInjuries <- arrange(deathsAndInjuries, desc(combinedDeathsAndInjuries))
head(deathsAndInjuries)
```

```
##        eventType FATALITIES INJURIES combinedDeathsAndInjuries
## 1        TORNADO       5633    91346                     96979
## 2 EXCESSIVE HEAT       1903     6525                      8428
## 3      TSTM WIND        504     6957                      7461
## 4          FLOOD        470     6789                      7259
## 5      LIGHTNING        816     5230                      6046
## 6           HEAT        937     2100                      3037
```

By combining the injuries with the fatalities of the data, we see that
tornadoes are clearly the most harmful to weather events in the data set.

### Question 2: Economic Consequences

First we collect the subsets of columns related to the economic impact
of events, namely crop and property damage.


```r
propertyData <- select(weatherData, EVTYPE, PROPDMG, PROPDMGEXP, CROPDMG, CROPDMGEXP)
```

While the impact on the human population from Question 1 certainly
also has economic consequences, they will not be taken into
consideration here because no financial information in these respects
was included in the dataset, making any attempt to include the "human"
element of the economic consequences purlely speculative.

Dealing with factor levels in `propertyData$CROPDMGEXP` and `propertyData$CROPDMGEXP`:


```r
levels(propertyData$PROPDMGEXP)
```

```
##  [1] ""  "+" "-" "0" "1" "2" "3" "4" "5" "6" "7" "8" "?" "B" "H" "K" "M"
## [18] "h" "m"
```

```r
levels(propertyData$CROPDMGEXP)
```

```
## [1] ""  "0" "2" "?" "B" "K" "M" "k" "m"
```

according to
[this site](https://rstudio-pubs-static.s3.amazonaws.com/58957_37b6723ee52b455990e149edde45e5b6.html)
the factor levels above can be cleaned up by uppercasing any lowercase
variables, making numbers 1-8 multipliers of ten (i.e a value of 1
multiplies the value in the corresponding DMG column by 10, just like
a value of 3, or 4, or etc.), the value `+` multiply by 1 and the
values `-` and `?` multiply by 0.

Only keep rows where `CROPDMG > 0` or `PROPDMG > 0`


```r
propertyData <- filter(propertyData, CROPDMG > 0 | PROPDMG > 0)
```

Now manually clean up `EVTYPE`s that _obviously_ belong together


```r
propertyData[grep('tstm.*wind|thunderstorm.*wind', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'THUNDERSTORM WIND'
propertyData[grep('FLASH.*FLOOD', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'FLASH FLOOD'
propertyData[grep('URBAN.*FLOOD|^FLOODING$RIVER.*FLOOD', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'FLOOD'
propertyData[grep('HURRICANE', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HURRICANE'
propertyData[grep('HIGH.*WIND|^WIND$STRONG.*WIND', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HIGH WIND'
propertyData[grep('WILD.*FIRE', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'WILDFIRE'
propertyData[grep('WINTER.*WEATHER|LIGHT SNOW', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'WINTER WEATHER'
propertyData[grep('FOG', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'DENSE FOG'
propertyData[grep('COASTAL.*FLOOD', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'COASTAL FLOOD'
propertyData[grep('HEAVY.*SURF', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HIGH SURF'
propertyData[grep('HEAVY.*RAIN', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HEAVY RAIN'
propertyData[grep('EXCESSIVE.*SNOW', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HEAVY SNOW'
propertyData[grep('EXTREME.*COLD', propertyData$EVTYPE, ignore.case = T),]$EVTYPE <- 'EXTREME COLD'
```

Now over approximately 98% of the data have been assigned to a proper
category outlined in
[here](file:///home/chris/Coursera/Reproducible%20Research/project2/repdata-peer2_doc-pd01016005curr.pdf)
in section 2.1.1 "Storm Data Event Table"


```r
evCounts <- as.data.frame(table(propertyData$EVTYPE))
evCounts <- arrange(evCounts, desc(Freq))
sum(evCounts$Freq[evCounts$Freq > 1000])
```

```
## [1] 239100
```

```r
nrow(propertyData)
```

```
## [1] 245031
```

```r
239100/245031
```

```
## [1] 0.9757949
```

## Results

##
