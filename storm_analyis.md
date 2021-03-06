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

```
## 
## Attaching package: 'dplyr'
## 
## The following objects are masked from 'package:stats':
## 
##     filter, lag
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
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

Now we can aggregate them by event type:


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
economicData <- select(weatherData, EVTYPE, PROPDMG, PROPDMGEXP, CROPDMG, CROPDMGEXP)
```

While the impact on the human population from Question 1 certainly
also has economic consequences, they will not be taken into
consideration here because no financial information in these respects
was included in the dataset, making any attempt to include the "human"
element of the economic consequences purlely speculative.

We only keep rows where `CROPDMG > 0` or `PROPDMG > 0`


```r
economicData <- filter(economicData, CROPDMG > 0 | PROPDMG > 0)
```

Now we manually merge (to the extent possible) `EVTYPE`s that are not
listed in section 2.1.1 of the
[NOAA Guide](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf)
with good matches that are.


```r
economicData[grep('tstm.*wind|thunderstorm.*wind', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'THUNDERSTORM WIND'
economicData[grep('FLASH.*FLOOD', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'FLASH FLOOD'
economicData[grep('URBAN.*FLOOD|^FLOODING$|RIVER.*FLOOD', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'FLOOD'
economicData[grep('HURRICANE', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HURRICANE'
economicData[grep('HIGH.*WIND|^WIND$|STRONG.*WIND', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HIGH WIND'
economicData[grep('WILD.*FIRE', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'WILDFIRE'
economicData[grep('WINTER.*WEATHER|LIGHT SNOW', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'WINTER WEATHER'
economicData[grep('FOG', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'DENSE FOG'
economicData[grep('COASTAL.*FLOOD', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'COASTAL FLOOD'
economicData[grep('HEAVY.*SURF', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HIGH SURF'
economicData[grep('HEAVY.*RAIN', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HEAVY RAIN'
economicData[grep('EXCESSIVE.*SNOW', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'HEAVY SNOW'
economicData[grep('EXTREME.*COLD', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'EXTREME COLD'
economicData[grep('STORM.*SURGE', economicData$EVTYPE, ignore.case = T),]$EVTYPE <- 'STORM SURGE/TIDE'
```

All events occuring more than 1000 times are now officially recognized
event categories, as outlined in the document mentioned above. This
is approximately 98% of all recorded events and will be considered a
sufficient base for the rest of this analysis.


```r
evCounts <- as.data.frame(table(economicData$EVTYPE))
evCounts <- arrange(evCounts, desc(Freq))
eventsOccurringMoreThan1000Times <- sum(evCounts$Freq[evCounts$Freq > 1000])
eventsOccurringMoreThan1000Times/nrow(economicData)
```

```
## [1] 0.9770764
```

Creating economic values from `CROPDMG`, `CROPDMGEXP`, `PROPDMG` and `PROPDMGEXP`:

Interpreting the factor levels was done with the help of section 2.7 of
the complementary
[NOAA document](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf)
as well as [this site](https://rstudio-pubs-static.s3.amazonaws.com/58957_37b6723ee52b455990e149edde45e5b6.html)
where we see how the factor levels not addressed in the NOAA document
can be interpreted. Basically values with numbers 1-8 are multipliers
of 10 (i.e a value of 1 multiplies the value in the corresponding DMG
column by 10, just like a value of 3, or 4, or etc.). The value `+`
multiplies by 1 and the values `-` and `?` multiply by 0.


```r
levels(economicData$PROPDMGEXP)
```

```
##  [1] ""  "+" "-" "0" "1" "2" "3" "4" "5" "6" "7" "8" "?" "B" "H" "K" "M"
## [18] "h" "m"
```

```r
levels(economicData$CROPDMGEXP)
```

```
## [1] ""  "0" "2" "?" "B" "K" "M" "k" "m"
```

Now we use the following helper function to interpret these
factors. Subsequently we aggregate the monetary damage by event type.


```r
multiplierFor <- function(type){
    type <- as.character(type)
    if(grepl('[a-zA-Z]', type)) type <- tolower(type)
    switch(type,
           '0' = 10,
           '1' = 10,
           '2' = 10,
           '3' = 10,
           '4' = 10,
           '5' = 10,
           '6' = 10,
           '7' = 10,
           '8' = 10,
           '-' = 0,
           '?' = 10,
           '+' = 1,
           'h' = 100,
           'k' = 1000,
           'm' = 1000000,
           'b' = 1000000000,
           0)
}

economicData <- economicData %>% rowwise() %>% mutate(monetaryDamage = PROPDMG*multiplierFor(PROPDMGEXP) + CROPDMG*multiplierFor(CROPDMGEXP))
aggregatedDamages <- aggregate(economicData$monetaryDamage, list(economicData$EVTYPE), sum)
colnames(aggregatedDamages) <- c('EVTYPE', 'monetaryValue')
aggregatedDamages <- arrange(aggregatedDamages, desc(monetaryValue))
head(aggregatedDamages)
```

```
##             EVTYPE monetaryValue
## 1            FLOOD  160756453460
## 2        HURRICANE   90271472810
## 3          TORNADO   57352117607
## 4 STORM SURGE/TIDE   47965579000
## 5             HAIL   18758224527
## 6      FLASH FLOOD   18439123353
```

Here we see that flooding produces the highest cost in damages in the dataset.

## Results


```r
library(ggplot2)
g <- ggplot(deathsAndInjuries[c(1:20),],
            aes(eventType, combinedDeathsAndInjuries, group=1))
g + geom_bar(stat='identity') +
    theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
    labs(title = "Top Twenty Events Causing Death or Injury", x = "Event Type", y = "Combined Deaths and Injuries")
```

![](storm_analyis_files/figure-html/unnamed-chunk-11-1.png) 

Here we can see that tornadoes are by a large margin the most significant weather source causing death and injuries.


```r
g <- ggplot(aggregatedDamages[c(1:20),], aes(EVTYPE, monetaryValue, group=1))
g + geom_bar(stat='identity') +
    theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
    labs(title = "Top Twenty Events Causing Economic Damage", x = "Event Type", y = "Monetary Value")
```

![](storm_analyis_files/figure-html/unnamed-chunk-12-1.png) 

In this table we see that flooding followed by hurricane damage are the most costly economic weather factors.
