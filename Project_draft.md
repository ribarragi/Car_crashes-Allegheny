Final project: Car crashes in Allegheny county
================
Ricardo Ibarra-Gil (ribarrag)
December 10, 2021

<style type="text/css">
  body{
  font-family: Helvetica;
  font-size: 12pt;
}
</style>

``` r
# import libraries and data set
library(readr)
library(tidyverse)
library(MASS)
library(corrplot)  
library(plotly)
library(knitr)
library(ggplot2)
crashes <- read_csv("2c13021f-74a9-4289-a1e5-fe0472c89881.csv")
```

### Introduction

This project looks into data about car crashes in Allegheny County. Its
main reserach purposes are:

**1. Provide insightful descriptive statistics about car crashes in
Allegheny county.** Present statistics and patterns in the data in a
clear and useful way for public policy decisions.

**2. Run a preliminary model.** Find factors that potentially explain a
target variable called ‘Severity’, that measures the gravity of each car
crash, to provide additional statistical analysis.

The ultimate goal of the project is to develop a document that could
serve as support for policy decisions regarding road safety. It could
assist policy makers to best allocate resources to diminish the negative
social impacts of car crashes.

### 1. About the data set

The original data set was accessed through the Western Pennsylvania
Regional Data Center[1] (see footnote for link to data set).

In its original format and content, the data set has 204898 rows and 191
columns. It comprises all car crashes registered in Allegheny County
from 2004 to 2020 (each row represents one accident). The data underwent
a cleaning and variable selection process that is detailed in the
following section of the document.

The document comprises four sections:

-   **Section 2** includes the cleaning and preprocessing of the data
    set, which also serves as a first glance into the data and its
    structure.
-   **Section 3** presents the Exploratory Descriptive Analysis that
    provides more in-depth insights.
-   **Section 4** covers a multinomial logistic regression model, to
    explore which factors contribute to the severity of car crashes.
-   Finally, **section 5** includes conclusions and future work.

### 2. Data cleaning

The variables of interest for this document were selected, and put into
5 subsets for preprocessing: crash consequences, driver conditions, time
and place of crash, raod and vehicles variables, and age of drivers:

``` r
# create subcategories with variables
crash_vars <- c('PROPERTY_DAMAGE_ONLY','INJURY', 'INJURY_COUNT', 'INJURY_OR_FATAL', 'BELTED_DEATH_COUNT', 'FATAL', 'FATAL_COUNT', 'PERSON_COUNT')
driver_conditions <- c('AGGRESSIVE_DRIVING', 'CELL_PHONE', 'DRINKING_DRIVER', 'DISTRACTED', 'FATIGUE_ASLEEP', 'ILLEGAL_DRUG_RELATED', 'RUNNING_RED_LT', 'RUNNING_STOP_SIGN', 'SPEEDING','TAILGATING', 'UNBELTED')
time_place <- c('DAY_OF_WEEK', 'HOUR_OF_DAY','ILLUMINATION_DARK', 'WEATHER')
road_vehicles <- c('CURVED_ROAD', 'ROAD_CONDITION', 'INTERSECTION', 'LOCAL_ROAD', 'URBAN_RURAL', 
                   'SCHOOL_BUS', 'SPEED_LIMIT', 'WET_ROAD', 'VEHICLE_COUNT')
driver_age <- c('DRIVER_16YR', 'DRIVER_17YR', 'DRIVER_18YR', 'DRIVER_19YR', 'DRIVER_20YR', 
                'DRIVER_50_64YR', 'DRIVER_65_74YR', 'DRIVER_75PLUS')
all_vars <- c(driver_conditions, time_place, crash_vars, driver_age, road_vehicles)
# car_crashes will be my main data set
crashes_df = subset(crashes, select = all_vars)
```

**1. Variables related to the results of a crash**. Like number of
injured, fatalities, if the crash resulted only in property damage,
among others.

``` r
# This is a brief function created to have only mean, min and max from the variables, and ignoring NAs, instead of having all quartiles as with a 'traditional' summary.
summary_functions <- function(i) {c(mean = mean(i,  na.rm = TRUE), min = min(i,  na.rm = TRUE), max = max(i,  na.rm = TRUE), stdv = sd(i, na.rm = TRUE)) } 
```

As basic as they can be, these descriptive statistics show interesting
information.

For instance, it can be seen that, on average, 53% of car crashes had
only property damage, while 44.69% had one or more people injured, and
0.54% had at least one casualty.

``` r
# This applies the defined function to only the subset of variables related to car crash results
kable(round(sapply(subset(crashes_df, select = crash_vars[1:4]), summary_functions), 3), results = 'asis')
```

|      | PROPERTY_DAMAGE_ONLY | INJURY | INJURY_COUNT | INJURY_OR_FATAL |
|:-----|---------------------:|-------:|-------------:|----------------:|
| mean |                0.530 |  0.447 |        0.610 |           0.450 |
| min  |                0.000 |  0.000 |        0.000 |           0.000 |
| max  |                1.000 |  1.000 |       44.000 |           1.000 |
| stdv |                0.499 |  0.497 |        0.858 |           0.498 |

The standard deviation for INJURY_COUNT tells us that, although there
are some observations with high values (like the max = 44), the vast
majority of the data for that variable is concentrated around its mean
(0.61 injured per car crash).

As for the min and max of these variables, since most of them are
dummies, they should be 0 and 1, except for the variables *count*, that
show counts of people in each accident.

``` r
# This applies the defined function to only the subset of variables related to car crash results
kable(round(sapply(subset(crashes_df, select = crash_vars[5:8]), summary_functions), 3), results = 'asis')
```

|      | BELTED_DEATH_COUNT | FATAL | FATAL_COUNT | PERSON_COUNT |
|:-----|-------------------:|------:|------------:|-------------:|
| mean |              0.001 | 0.005 |       0.006 |        2.426 |
| min  |              0.000 | 0.000 |       0.000 |        0.000 |
| max  |              2.000 | 1.000 |       3.000 |       99.000 |
| stdv |              0.031 | 0.073 |       0.081 |        1.850 |

For example, PERSON_COUNT is how many people were involved in the
accident. It has a standard deviation of 1.85, which indicates that most
observations are concentrated around its average. Its maximum value of
99 is already a redflag in our data set: it is not very probable that
there was a crash involving 99 people, so this can be an outlier that
can even be an encoding error. To further examine this data, its boxplot
is graphed:

``` r
# create a box plotw with ggplot 
ggplot(crashes_df, aes(y=PERSON_COUNT)) + geom_boxplot(color = 'steelblue') + 
  labs(title = 'Graph 1. Box plot for count of people involved in accident', y = 'Person Count') +
  theme(plot.title = element_text(hjust = 0.5, # I will use Helvetica bold  for titles of graphs and centered titles
                                  size = 16, 
                                  family = 'Helvetica-Bold'), 
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        plot.margin = margin(20, 20, 15, 15),
        panel.background = element_rect(fill = 'white'), # set the back of the graph to white
        panel.grid.major.y = element_line(size = .08, # leave only thin dashed lines
                                          linetype = 'dashed', 
                                          color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Boxplot Person Count-1.png" title="Boxplot. Count of persons involved in crash" alt="Boxplot. Count of persons involved in crash" style="display: block; margin: auto;" />

``` r
crashes_df$PERSON_COUNT[crashes_df$PERSON_COUNT > 20] = NA # this is setting values above 20 to missing values
```

The boxplot in **Graph 1** helps define the presence of outliers. In
this case, a decision about our data had to be made: outliers in the
variable PERSON_COUNT will be considered those crashes in which more
than 20 individuals were involved, and in this case were set to NA. As
mentioned, some can even be data entry errors.

As can be seen in the table below, all variables in this subset are
numerical. Most of them are dummies (indicate the presence/absence of an
attribute), and some are ordinal variables (count the number of times
any attribute was present in a car crash).

``` r
str(subset(crashes_df, select = crash_vars))  #asks for the structure of the variables in the category crash_vars
```

    ## tibble [204,898 × 8] (S3: tbl_df/tbl/data.frame)
    ##  $ PROPERTY_DAMAGE_ONLY: num [1:204898] 1 0 0 0 1 1 0 1 0 1 ...
    ##  $ INJURY              : num [1:204898] 0 1 1 1 0 0 1 0 1 0 ...
    ##  $ INJURY_COUNT        : num [1:204898] 0 1 1 1 0 0 3 0 1 0 ...
    ##  $ INJURY_OR_FATAL     : num [1:204898] 0 1 1 1 0 0 1 0 1 0 ...
    ##  $ BELTED_DEATH_COUNT  : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ FATAL               : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ FATAL_COUNT         : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ PERSON_COUNT        : num [1:204898] 1 1 1 3 1 2 5 1 2 2 ...

From these variables, a **‘SEVERITY’** variable was created to represent
how bad the crash was in terms of damages and injured people. The
severity variable is classified according to the following criteria:

-   0 = No property damage, no injured, no death  
-   1 = Only property damage  
-   2 = One injured, no death  
-   3 = Several injured, no death  
-   4 = Death, can be one or multiple

This variable will serve as dependant varable for the regression
analysis of section 4.

``` r
# create the severity variable according to the description in the text
crashes_df <- crashes_df %>% 
  mutate(SEVERITY = if_else((PROPERTY_DAMAGE_ONLY == 0) & (INJURY == 0) & (INJURY_COUNT == 0) & (FATAL == 0), 0,
        + ifelse ((PROPERTY_DAMAGE_ONLY == 1) & (INJURY == 0) & (INJURY_COUNT == 0), 1,
        + ifelse ((INJURY == 1) & (INJURY_COUNT == 1) & (FATAL == 0) & (FATAL_COUNT == 0), 2, 
        + ifelse ((INJURY == 1) & (INJURY_COUNT > 1)  & (FATAL == 0) & (FATAL_COUNT == 0), 3,
        + ifelse ((FATAL == 1), 4, NA))))))
crashes_df$SEVERITY_L <- factor(crashes_df$SEVERITY,
                                 levels = c(0, 1, 2, 3, 4),
                                 labels = c("(0) No Damages", "(1) Only property damages", "(2) Only 1 injured", '(3) Several injured, no deaths', '(4) Fatalities (1 or many)'))
```

The mean of SEVERITY is 1.557. This indicates that the consequence of
the average car crash in Allegheny county from 2004 to 2020 lies between
only property damage (1), and 1 person injured with no fatalities (2).

However, it is also interesting to the see the numbers in absolute
terms:

``` r
# display the categories of var SEVERITY with its frequency
kable(table(crashes_df$SEVERITY_L), col.names = c('Severity of crash', 'Frequency'))
```

| Severity of crash                | Frequency |
|:---------------------------------|----------:|
| \(0\) No Damages                 |      3305 |
| \(1\) Only property damages      |    108590 |
| \(2\) Only 1 injured             |     68450 |
| \(3\) Several injured, no deaths |     22658 |
| \(4\) Fatalities (1 or many)     |      1099 |

In the table above it can fully appreciated that a very high number of
crashes result in only property damage (108590 out of 204102). Also,
that there were 22658 crashes that resulted in several people injured,
and 1099 that resulted in one or more fatalities.

**2. Variables related to the drivers’ conditions**. The following
variables are all dummies (0,1), so it is more interesting to show the
mean for each of them; but also the min and max to verify for any
potential missing values encoded as numeric:

``` r
# I separated the summary of the variables in driving conditions, so they appear in two different rows, and not all 11 in one single row, because it conflicts with margins
kable(round(sapply(subset(crashes_df, select = c(driver_conditions[1:5])), summary_functions), 3))
```

|      | AGGRESSIVE_DRIVING | CELL_PHONE | DRINKING_DRIVER | DISTRACTED | FATIGUE_ASLEEP |
|:-----|-------------------:|-----------:|----------------:|-----------:|---------------:|
| mean |              0.509 |      0.011 |           0.088 |      0.125 |          0.014 |
| min  |              0.000 |      0.000 |           0.000 |      0.000 |          0.000 |
| max  |              1.000 |      1.000 |           1.000 |      1.000 |          1.000 |
| stdv |              0.500 |      0.106 |           0.284 |      0.331 |          0.116 |

``` r
kable(round(sapply(subset(crashes_df, select = c(driver_conditions[6:11])), summary_functions), 3))
```

|      | ILLEGAL_DRUG_RELATED | RUNNING_RED_LT | RUNNING_STOP_SIGN | SPEEDING | TAILGATING | UNBELTED |
|:-----|---------------------:|---------------:|------------------:|---------:|-----------:|---------:|
| mean |                0.010 |          0.034 |             0.016 |    0.043 |      0.051 |    0.119 |
| min  |                0.000 |          0.000 |             0.000 |    0.000 |      0.000 |    0.000 |
| max  |                1.000 |          1.000 |             1.000 |    1.000 |      1.000 |    1.000 |
| stdv |                0.098 |          0.182 |             0.126 |    0.202 |      0.221 |    0.324 |

It can be seen that there are no variables with maximum greater than 1
(which is correct, since all are dummies). It is interesting that in a
bit more than half of the accidents, there was some AGGRESSIVE_DRIVING
behavior; in 12.53% there was at least one distracted driver, and in
11.94% of all crashes, at least one of the drivers involved in the
accident was not wearing a set belt.

As can be seen below, all variables are numeric.

``` r
# print structure of variables in the driver_conditions subset
str(subset(crashes_df, select = c(driver_conditions)))
```

    ## tibble [204,898 × 11] (S3: tbl_df/tbl/data.frame)
    ##  $ AGGRESSIVE_DRIVING  : num [1:204898] 0 0 1 1 0 0 1 0 1 1 ...
    ##  $ CELL_PHONE          : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ DRINKING_DRIVER     : num [1:204898] 1 0 0 0 0 0 0 0 0 0 ...
    ##  $ DISTRACTED          : num [1:204898] 1 0 0 0 0 0 0 0 0 0 ...
    ##  $ FATIGUE_ASLEEP      : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ ILLEGAL_DRUG_RELATED: num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ RUNNING_RED_LT      : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ RUNNING_STOP_SIGN   : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ SPEEDING            : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ TAILGATING          : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ UNBELTED            : num [1:204898] 1 0 0 0 0 1 1 0 0 0 ...

The dataset has dummy variables for certain age cohorts, but
unfortunately not for all.

``` r
#same separation of variables descriptors to fit the margins
kable(round(sapply(subset(crashes_df, select = c(driver_age[1:4])), summary_functions),3))
```

|      | DRIVER_16YR | DRIVER_17YR | DRIVER_18YR | DRIVER_19YR |
|:-----|------------:|------------:|------------:|------------:|
| mean |       0.013 |       0.033 |       0.040 |       0.043 |
| min  |       0.000 |       0.000 |       0.000 |       0.000 |
| max  |       2.000 |       1.000 |       1.000 |       1.000 |
| stdv |       0.114 |       0.179 |       0.197 |       0.203 |

``` r
kable(round(sapply(subset(crashes_df, select = c(driver_age[5:8])), summary_functions),3))
```

|      | DRIVER_20YR | DRIVER_50_64YR | DRIVER_65_74YR | DRIVER_75PLUS |
|:-----|------------:|---------------:|---------------:|--------------:|
| mean |       0.045 |          0.291 |          0.090 |         0.064 |
| min  |       0.000 |          0.000 |          0.000 |         0.000 |
| max  |       1.000 |          1.000 |          1.000 |         1.000 |
| stdv |       0.207 |          0.454 |          0.286 |         0.244 |

From a first glance at the data we can see that there is an anomaly with
the dummy DRIVER_16YR (it indicates if there was (or not) at least one
driver of age 16 in the car crash). This dummy should not have values
greater than 1. So, the values that are greater than 1 (which are
exactly: 2, and both set to 2) will be set to missing values.

As can be seen below. All these variables have only numeric values only:

``` r
# set observations that are 2 to NA because variable is dummy (0,1 only)
crashes_df$DRIVER_16YR[crashes_df$DRIVER_16YR == 2]= NA
str(subset(crashes_df, select = c(driver_age)))
```

    ## tibble [204,898 × 8] (S3: tbl_df/tbl/data.frame)
    ##  $ DRIVER_16YR   : num [1:204898] 0 0 0 0 1 0 0 0 0 0 ...
    ##  $ DRIVER_17YR   : num [1:204898] 0 0 0 0 0 0 0 0 0 1 ...
    ##  $ DRIVER_18YR   : num [1:204898] 0 0 0 0 0 0 0 0 1 0 ...
    ##  $ DRIVER_19YR   : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ DRIVER_20YR   : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ DRIVER_50_64YR: num [1:204898] 0 1 0 1 0 0 1 0 0 1 ...
    ##  $ DRIVER_65_74YR: num [1:204898] 0 0 0 1 0 1 0 0 0 0 ...
    ##  $ DRIVER_75PLUS : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...

**3. Time and place of the accidents.** This category contains only a
few variables The table below indicates that one of them (HOUR_OF_DAY)
is of type character.

``` r
# print structure of variables in the time_place subset
str(subset(crashes_df, select = c(time_place)))
```

    ## tibble [204,898 × 4] (S3: tbl_df/tbl/data.frame)
    ##  $ DAY_OF_WEEK      : num [1:204898] 5 5 5 6 5 6 6 7 7 1 ...
    ##  $ HOUR_OF_DAY      : chr [1:204898] "02" "10" "18" "09" ...
    ##  $ ILLUMINATION_DARK: num [1:204898] 1 0 1 0 1 0 0 0 0 0 ...
    ##  $ WEATHER          : num [1:204898] 1 1 1 1 1 1 1 1 2 2 ...

It stands out that the WEATHER variable has a high max:

``` r
kable(round(sapply(subset(crashes_df, select = c('DAY_OF_WEEK', 'ILLUMINATION_DARK', 'WEATHER')), summary_functions), 3))
```

|      | DAY_OF_WEEK | ILLUMINATION_DARK | WEATHER |
|:-----|------------:|------------------:|--------:|
| mean |       4.150 |             0.338 |   1.880 |
| min  |       1.000 |             0.000 |   1.000 |
| max  |       7.000 |             1.000 |  99.000 |
| stdv |       1.963 |             0.473 |   4.223 |

This high max value of WEATHER should be explored in more detail. These
are the values taken by this variable:

``` r
# show the unique entries of weather
unique(crashes_df$WEATHER)
```

    ##  [1]  1  2  4  8  3  6  9  5  7 NA 10 98 99

The NAs will be dropped at the end of this section, but there are some
98’s and 99’s, and according to the documentation, there are only 10
valid values. So I will also set the observations above 10 as missing
values (there are not many, only 0.017% of total values are 98 or 99).

``` r
# set weather variable observations greater than 10 to NA
crashes_df$WEATHER[crashes_df$WEATHER > 10] = NA
```

And the variable is set into factor, and labels are defined for each
level.

``` r
crashes_df$WEATHER <- factor(crashes_df$WEATHER,
                              levels = c(1, 2, 3, 4, 5, 6, 7, 8, 9, 10),
                              labels = c("Blowing sand / soil", "Blowing snow", "Clear", 'Cloudy', 'Fog/Smoke', 'Freezing rain', 'Rain', 'Severe crosswinds', 'Sleet / hail', 'Snow'))
```

Something similar is happening with the variable HOUR_OF_DAY. From
documentation (and the fact that there are only 24 hours in a day) we
know that it can only go from 00 to 23. However, the variable has some
99s in it, which will be set to missing values:

``` r
unique(crashes_df$HOUR_OF_DAY)
```

    ##  [1] "02" "10" "18" "09" "22" "14" "08" "12" "11" "16" "19" "04" "99" "03" "06"
    ## [16] "15" "05" "17" "13" "07" "21" "23" "20" "01" "00" NA

``` r
crashes_df$HOUR_OF_DAY[crashes_df$HOUR_OF_DAY == 99] = NA
```

And it will also be set to factor, and set the label for its levels.

``` r
crashes_df$DAY_OF_WEEK <- factor(crashes_df$DAY_OF_WEEK,
                                   levels = c(1, 2, 3, 4, 5, 6, 7),
                                   labels = c("Sunday", "Monday", "Tuesday", 'Wednesday', 'Thursday', 'Friday', 'Saturday'))
```

**4. Variables about rod and vehicles.** The final of the data
categories.

``` r
str(subset(crashes_df, select = c(road_vehicles)))
```

    ## tibble [204,898 × 9] (S3: tbl_df/tbl/data.frame)
    ##  $ CURVED_ROAD   : num [1:204898] 0 1 0 0 0 0 0 0 0 0 ...
    ##  $ ROAD_CONDITION: num [1:204898] 0 6 0 1 0 1 0 0 1 1 ...
    ##  $ INTERSECTION  : num [1:204898] 0 0 1 1 0 1 0 0 0 1 ...
    ##  $ LOCAL_ROAD    : num [1:204898] 0 1 0 1 1 1 1 0 0 1 ...
    ##  $ URBAN_RURAL   : num [1:204898] 3 3 3 3 3 3 3 3 3 3 ...
    ##  $ SCHOOL_BUS    : num [1:204898] 0 0 0 0 0 0 0 0 0 0 ...
    ##  $ SPEED_LIMIT   : num [1:204898] 35 35 35 25 20 25 30 40 40 25 ...
    ##  $ WET_ROAD      : num [1:204898] 0 0 0 1 0 1 0 0 1 1 ...
    ##  $ VEHICLE_COUNT : num [1:204898] 1 1 1 2 2 2 3 1 2 2 ...

All of the variables are numeric, and their summary statistics are:

``` r
kable(round(sapply(subset(crashes_df, select = c(road_vehicles)[1:5]), summary_functions), 3))
```

|      | CURVED_ROAD | ROAD_CONDITION | INTERSECTION | LOCAL_ROAD | URBAN_RURAL |
|:-----|------------:|---------------:|-------------:|-----------:|------------:|
| mean |       0.179 |          0.882 |        0.435 |      0.584 |       2.950 |
| min  |       0.000 |          0.000 |        0.000 |      0.000 |       1.000 |
| max  |       1.000 |         99.000 |        1.000 |      1.000 |       3.000 |
| stdv |       0.383 |          2.600 |        0.496 |      0.493 |       0.311 |

``` r
kable(round(sapply(subset(crashes_df, select = c(road_vehicles)[6:9]), summary_functions), 3))
```

|      | SCHOOL_BUS | SPEED_LIMIT | WET_ROAD | VEHICLE_COUNT |
|:-----|-----------:|------------:|---------:|--------------:|
| mean |      0.008 |      34.637 |    0.182 |         1.831 |
| min  |      0.000 |       0.000 |    0.000 |         0.000 |
| max  |      1.000 |      75.000 |    1.000 |        14.000 |
| stdv |      0.087 |      11.380 |    0.385 |         0.770 |

We can see that ROAD_CONDITION has max values of 98 and 99, which
doesn’t make much sense

``` r
unique(crashes_df$ROAD_CONDITION)
```

    ##  [1]  0  6  1  5  3  8  4  2  7  9 22 99 98 NA

After checking the data set documentation,98 and 99 from ROAD_CONDITION
are reclassified as NA, and 8, 9 and 22 into ‘Other’.

``` r
crashes_df$ROAD_CONDITION[crashes_df$ROAD_CONDITION > 22] = NA
crashes_df$ROAD_CONDITION <- factor(crashes_df$ROAD_CONDITION,
                                      levels = c(0, 1, 2, 3, 4, 5, 6, 7, c(8, 9, 22)),
                                      labels = c("Dry", "Wet", "Sand / mud / gravel", 'Snow', 'Slush', 'Ice', 'Ice patches', 'Water', 'Other', 'Other', 'Other'))
```

Now that we have taken a glance at the data, and made all changes to
clean it, the missing values are dropped, and the data is ready to start
analyzing it in more depth:

``` r
crashes_df <- crashes_df %>% drop_na()
```

I have a data set clean and ready to work with. Also, now I have a
pretty good idea of what is in my data set

### 3. Exploratory Descriptive Analysis

A first interesting insight about car crashes in Allegheny County is how
they are distributed in time.

``` r
ggplot(crashes_df, aes(x=(HOUR_OF_DAY)))+
  geom_bar(stat="count", width=0.7, fill="steelblue") +
  labs(title = 'Graph 2. Car crashes per hour of the day', y = 'Crashes count', x = 'Hour of the day') + 
  scale_y_continuous(breaks=seq(0,14000,2000)) +
  theme(plot.title = element_text(hjust = 0.5, 
                                size = 16, 
                                family = 'Helvetica-Bold'),
      axis.text = element_text(size = 10), 
      axis.title.x = element_text(size = 14, 
                                  margin = margin(5, 0, 0, 0)),
      axis.title.y = element_text(size = 14, 
                                  margin = margin(0, 5, 0, 0)),
      axis.text.x = element_text(margin = margin(0, 0, 0, 0)),
      axis.ticks = element_blank(),
      plot.margin = margin(20, 20, 15, 15), 
      panel.background = element_rect(fill = 'white'),
      panel.grid.major.y = element_line(size = .08, 
                                        linetype = 'dashed', 
                                        color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Graph Hours-1.png" title="Car crashes per hour of day" alt="Car crashes per hour of day" style="display: block; margin: auto;" />

It can be seen in **Graph 2** that, in absolute terms, most crashes
happen in the late afternoon-early evening, from 3:00 to 5:00 PM., with
a peak at 17:00 hours. This is interesting and a bit surprising.

It cluld be expected that more accidents happened towards 19:00 when (I
believe) people leave their jobs and go home. A possible explanation
could be that before peak hour, people try to get home quickly before
traffic starts, and drive with less care, so there is an increased
number of accidents around 5:00 PM.

**Graph 3** shows the relative distribution of crashes with fatal
outcomes (one or more people died in the accident).

``` r
barplot(tapply(crashes_df$FATAL, list(crashes_df$HOUR_OF_DAY), mean), main = 'Graph 3. Average crashes with fatal outcomes per hour of the day', xlab = 'Hour of the day', ylab = 'Avg crashes with fatalities', col = 'steelblue', las = 2, ylim=c(0,.02))
```

<img src="Project_draft_files/figure-gfm/Avg crashes with fatal by hour-1.png" style="display: block; margin: auto;" />

It should be noticed from **Graph 3** that the y axis has a range of low
numbers because the number of crashes with casualties is relatively
small (as seen in Section 2) as compared to the total number of car
accidents. Nevertheless, we can still observe that crashes with deaths
concentrate around the first hours of the day (00:00 and 5:00 am) and
have a peak at 02:00 am.

``` r
ggplot(crashes_df, aes(x=(DAY_OF_WEEK)))+
  geom_bar(stat="count", width=0.7, fill = "steelblue") +
  labs(title = 'Graph 4. Car crashes per day of the week', y = 'Crashes count', x = 'Day of the week') + 
  scale_y_continuous(breaks=seq(20000,32000,2000)) +
  coord_cartesian(ylim = c(20000, 32000)) +
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold'),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(10, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(10, 0, 0, 0)),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.y = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Crashes by day of week-1.png" style="display: block; margin: auto;" />

When we turn our analysis to day of the week, it can be observed
(**Graph 4**) that Sunday is the day when less crashes happen (11.90% of
total crashes), which is perhaps associated with the fact that a lot of
people rest on that day, and don’t use their cars. Then crashes go up
during the week and reach a peak on Fridays (16.74% of total crashes).

``` r
# create a proportions table for the graph
data_plotly <- prop.table(table(crashes_df$DAY_OF_WEEK, crashes_df$HOUR_OF_DAY))
# Here, i convert the column names (hours) into format 00:00, because otherwise they are presented like numbers (0, 1, 2, ...) and it is not clear that these are hours. 
colnames(data_plotly) <- format(strptime(as.numeric(colnames(data_plotly)), format = "%H"), format = "%H:%M")
# Transpose the matrix because im interested in changing the column names
data_plotly <- t(as.matrix(data_plotly))
# change column names to name of day
colnames(data_plotly)[1] <- 'SUN'
colnames(data_plotly)[2] <- 'MON'
colnames(data_plotly)[3] <- 'TUE'
colnames(data_plotly)[4] <- 'WED'
colnames(data_plotly)[5] <- 'THU'
colnames(data_plotly)[6] <- 'FRI'
colnames(data_plotly)[7] <- 'SAT'
data_plotly <- round(data_plotly*100,2)
# finally graph that information
p <- plot_ly(x = colnames(data_plotly), y = rownames(data_plotly), 
             z = data_plotly, 
             type = "heatmap", 
             colorscale= "RdBu", # set colors to from red to blue
             showscale = F) %>%
  layout(title = list(text ='Graph 5. Car crashes per time and day (% of car crashes)', font = list(family = "Helvetica-Bold", size = 16, color = 'black')), # had to change the order to show hours of day in increasing order
         yaxis = list(dtick = 1, ticks = '', ticklabelposition = "inside", autorange = 'reversed'), 
         xaxis = list(side = 'top', tickangle = 0),
         margin = list(l = 40, r = 40, b = 40, t = 80))
p
```

<img src="Project_draft_files/figure-gfm/Plotly graph Day Time-1.png" style="display: block; margin: auto;" />

An analysis of both day of the week and time of the day is displayed in
**Graph 5**. It is an interactive graph, so the user can hover over it,
and it will display the Day, Time and Percentage of crashes. The color
of the square indicates in what day and time when most crashes occur
(blue for less occurrences and red for more). For instance, on a Friday
at 17:00 hrs., 1.29% of all crashes of Allegheny County take place.

Moving towards other interesting variables, **Graph 6** displays the
percentage of crashes that had fatalities, by type of weather.

``` r
ggplot(crashes_df, aes(x = WEATHER, y = FATAL*100)) + 
  geom_bar(stat = "summary", fun = "mean",  width=0.7, fill="steelblue") + 
  theme(axis.text.x = element_text(angle = 90)) +
  labs(title = 'Graph 6. Percentage of crashes with fatalities per type of weather', y = 'Percentage', x = 'Weather') + 
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold', 
                                  margin = margin(0, 0, 15, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0)),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.y = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Crashes with fatalities by weather-1.png" style="display: block; margin: auto;" />

In almost 2.4% of crashes where there was sleet or hail, there was at
least one fatality. This compares, for example, with around 0.5% of
crashes with fatalities out of total crashes when there is sand or soil,
snow, when the sky is clear, or when there are severe crosswinds. This
should certainly be of interest for authorities to invest more in
training people to drive during this type of weather in order to reduce
car crash fatalities.

**Graph 7** shows the average severity of crashes by road condition,
separating between speeding and not speeding. Let’s remeber that
severity is a variable that goes from 0 to 4 and measures the negative
impact of a car crash, so the higher the mean, the most negative the
consequence of a crash.

``` r
ggplot(crashes_df, aes(x = ROAD_CONDITION, y = SEVERITY)) + geom_bar(stat = "summary", fun = "mean") + facet_grid(crashes_df$SPEEDING) + # here, the facet dividing over speeding
  scale_y_continuous(breaks=seq(1,2,0.4)) + coord_cartesian(ylim = c(1.2, 1.8)) +
  geom_bar(stat = "summary", fun = "mean", fill="steelblue") +
  labs(title = 'Graph 7. Avg Severity of crashes per Road Condition and Speeding', y = 'Avg. Severity', x = 'Road Condition') + 
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold', 
                                  margin = margin(0, 0, 15, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0), angle = 90),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.y = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Severity by Road Condition and Speeding-1.png" style="display: block; margin: auto;" />

The lower part of **Graph 7** is when there was speeding in the
accident, and it can be noticed that severity of crashes increases
realtively to the upper part of the graph (no speeding) for almost all
road conditions, but especially for snow and slush. This could also be
of interest for policymakers: targeting speeding might have an impact on
reducing severity of car crashes in Allegheny county.

``` r
ages <- sapply(subset(crashes_df, select = c(driver_age)), sum)
age <- as.data.frame(ages)
```

The data set had dummy variables for certain age cohorts. Unfortunately,
there is not a variable with all driver’s age, but only for those whose
age fall in the cohorts displayed in the following graph:

``` r
ggplot(age, aes(x = rownames(age), ages)) + geom_bar(stat='identity', width=0.7, fill="steelblue") +
  labs(title = 'Graph 8. Crashes by age', 
       subtitle = '(available age cohorts only)', 
       x = 'Age cohorts', 
       y = 'Crashes count') +
   ylim(0, 60000) +
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold'),
        plot.subtitle = element_text(hjust = 0.5, 
                                     size = 12, 
                                     family = 'Helvetica', 
                                     margin = margin(0, 0, 5, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0)),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.y = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black')) +
  scale_x_discrete(labels = c('DRIVER_16YR' = '16','DRIVER_17YR' = '17',
                              'DRIVER_18YR' = '18','DRIVER_19YR' = '19',
                              'DRIVER_20YR' = '20','DRIVER_50_64YR' = '50-64',
                              'DRIVER_65_74YR' = '65-75','DRIVER_75PLUS' = '75'))
```

<img src="Project_draft_files/figure-gfm/Crashes by age-1.png" style="display: block; margin: auto;" />

Regarding age, it would be very interesting to analyze if being a young
driver is related to more severity of car crashes, compared to other age
cohorts in the regression model. So the variable YOUNG_DRIVER is
generated, and it is set to 1 if the driver is 16, 17, 18, 19 or 20, and
set to 0 otherwise.

``` r
crashes_df <- crashes_df %>% 
  mutate(YOUNG_DRIVER = if_else((DRIVER_16YR == 1) | (DRIVER_17YR == 1) | (DRIVER_18YR == 1) | (DRIVER_19YR == 1) | (DRIVER_20YR == 1), 1, 0))
```

Once we have this variable of YOUNG_DRIVERS, it is interesting to
analyze if, among young drivers there are some who are more prone to
accidents.

``` r
# only took the first five ages (the young drivers from 'ages')
young_pct <- round((ages[1:5]) *100 / sum(ages[1:5]),2)
young_pct <- as.data.frame(young_pct)
young_pct$`young_cohort` <- rownames(young_pct)
# added a column named prop, to get the folowing graph to work
young_pct <- young_pct %>% 
  mutate(Prop = "Proportions")
```

``` r
ggplot(young_pct, aes(x = Prop, y = young_cohort, fill = young_cohort)) +
  geom_col()+
  geom_text(aes(label = young_pct),
            position = position_stack(vjust = 0.5)) + coord_flip() + 
  scale_fill_manual(values=c('#3C6EC2', '#4E84C4', '#56B4E9', '#52854C', '#C3D7A4')) +
  labs(title = 'Graph 9. Young driver crashes: percentage of each age cohort', 
       x = '', 
       y = '') +
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold', 
                                  margin = margin(10, 0, -40, 0)), 
        panel.background = element_rect(fill = 'white'), legend.position = 'bottom',
        legend.title = element_blank(), legend.text=element_text(size = 10),
        legend.box.margin = margin(-80, 0, 0, 0),
        axis.text = element_blank(), 
        axis.ticks = element_blank()) +
  guides(fill = guide_legend(reverse = TRUE))
```

<img src="Project_draft_files/figure-gfm/Young drivers proportions graph-1.png" style="display: block; margin: auto;" />
As **Graph 9 shows**, of all **young** drivers (100%) who were involved
in a crash, 25.8% were aged 20 years old, while only 7.4% were 16 years
old. This can be partially explained by the fact that there might be
more 20 year old drivers than 15 year old drivers, but it is still an
interesting graph.

Also about young drivers, we can observe in **Graph 10** the average
severity of accidents (represented in the bars) by weather and separated
in two grids. The lower one for young drivers, and the upper one for
non-young drivers.

``` r
ggplot(crashes_df, aes(x = WEATHER, y = SEVERITY)) + geom_bar(stat = "summary", fun = "mean") + facet_grid(crashes_df$YOUNG_DRIVER) + # the facet is dividing over youn_driver variable here
  scale_y_continuous(breaks=seq(1,2,0.4)) + coord_cartesian(ylim = c(1.2, 1.8)) +
  geom_bar(stat = "summary", fun = "mean", fill="steelblue") +
  labs(title = 'Graph 10. Avg Severity of crashes per weather and Young Driver', y = 'Avg. Severity', x = 'Road Condition') + 
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold', 
                                  margin = margin(0, 0, 15, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0), angle = 90),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.y = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black'))
```

<img src="Project_draft_files/figure-gfm/Severity by weather and young driver-1.png" style="display: block; margin: auto;" />

From **Graph 10** we can say that the severity of accidents is higher
among young drivers (lower grid), particularly when there is freezing
rain, severe crosswinds and sleet / hail, compared to non-young drivers
(upper grid). Maybe a policy to educate young drivers to drive better
under those weather conditions could have a meaningful positive impact.

We have analyzed time and weather for crashes and fatalities. For this
last part of the Exploratory Data Analysis, there are two graphs about
drivers’ profile and behavior.

``` r
# I create the input for the following graph, needed tables for the subset driver conditions (their means as percentage)
crash_driver <- round((sapply(subset(crashes_df, select = c(driver_conditions)), mean)*100),1)
crash_driver_type <- as.data.frame(crash_driver)
crash_driver_type$`driver type` <- rownames(crash_driver_type)
fatalities <- subset(crashes_df, crashes_df$FATAL == 1)
fatal_driver <- round((sapply(subset(fatalities, select = c(driver_conditions)), mean)*100),1)
fatal_driver_type <- as.data.frame(fatal_driver)
fatal_driver_type$`driver type` <- rownames(fatal_driver_type)
```

``` r
ggplot(crash_driver_type, aes(x = reorder(`driver type`, crash_driver), y = crash_driver, label = crash_driver)) + 
  geom_point(stat='identity', color="steelblue", size = 12) +
  geom_segment(aes(y = 0, 
                   x = `driver type`, 
                   yend = crash_driver, 
                   xend = `driver type`), 
               color = "steelblue") +
  geom_text(color="white", size = 3.6) + # set color and size for text inside circles
  labs(title="Graph 11. Crashes by driver type", 
       subtitle="(% of crashes with at least one of these drivers)",
       x = 'Type of driver',
       y = '% of crashes') + 
  ylim(0, 60) + # set limits to y axis
  coord_flip() + # revert the fgraph so it appears from left to right instead of from down to up
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold'),
        plot.subtitle = element_text(hjust = 0.5, 
                                     size = 12, 
                                     family = 'Helvetica', 
                                     margin = margin(0, 0, 5, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0)),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 20, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.x = element_line(size = .08, # for almost all graphs I set the background to white 
                                          linetype = 'dashed', # but kept some thins dashed lines
                                          color = 'black')) +
        scale_x_discrete(labels = c('AGGRESSIVE_DRIVING' = 'AGGRESSIVE','DRINKING_DRIVER' = 'DRINKING',
                                    'RUNNING_RED_LT' = 'RUNNING RED LIGHT','RUNNING_STOP_SIGN' = 'RUNNING STOP SIGN',
                                    'FATIGUE_ASLEEP' = 'TIRED','CELL_PHONE' = 'DISTRACTED ON PHONE',
                                    'ILLEGAL_DRUG_RELATED' = 'ON ILLEGAL DRUG')) 
```

<img src="Project_draft_files/figure-gfm/Crashes by driver type-1.png" style="display: block; margin: auto;" />
**Graph 11** shows that 51.32% of crashes had at leat one aggressive
driver, by far the most common occurrence. This could hep policy makers
to implement measures to decrease stress and anger in the road. Second
comes distracted drivers.

``` r
ggplot(fatal_driver_type, aes(x=reorder(`driver type`, fatal_driver), y = fatal_driver, label = fatal_driver)) + 
  geom_point(stat='identity', 
             color = "steelblue", 
             size = 12) +
  geom_segment(aes(y = 0, 
                   x = `driver type`, 
                   yend = fatal_driver, 
                   xend = `driver type`), 
               color = "steelblue") +
  geom_text(color="white", size = 3.6) +
  labs(title = 'Graph 12. Crashes with fatalities by driver type', 
       subtitle = '(% of crashes with fatalities with at least one of these drivers)', 
       x = 'Car crashes with fatalities', 
       y = '% of crashes with fatalities') + 
  ylim(0, 60) +
  coord_flip() +
  theme(plot.title = element_text(hjust = 0.5, 
                                  size = 16, 
                                  family = 'Helvetica-Bold'),
        plot.subtitle = element_text(hjust = 0.5, 
                                     size = 12, 
                                     family = 'Helvetica', 
                                     margin = margin(0, 0, 5, 0)),
        axis.text = element_text(size = 10), 
        axis.title.x = element_text(size = 14, 
                                    margin = margin(5, 0, 0, 0)),
        axis.title.y = element_text(size = 14, 
                                    margin = margin(0, 5, 0, 0)),
        axis.text.x = element_text(margin = margin(0, 0, 0, 0)),
        axis.ticks = element_blank(),
        plot.margin = margin(20, 30, 15, 15), 
        panel.background = element_rect(fill = 'white'),
        panel.grid.major.x = element_line(size = .08, 
                                          linetype = 'dashed', 
                                          color = 'black')) +
  scale_x_discrete(labels = c('AGGRESSIVE_DRIVING' = 'AGGRESSIVE','DRINKING_DRIVER' = 'DRINKING',
                              'RUNNING_RED_LT' = 'RUNNING RED LIGHT','RUNNING_STOP_SIGN' = 'RUNNING STOP SIGN',
                              'FATIGUE_ASLEEP' = 'TIRED','CELL_PHONE' = 'DISTRACTED ON PHONE',
                              'ILLEGAL_DRUG_RELATED' = 'ON ILLEGAL DRUG')) 
```

<img src="Project_draft_files/figure-gfm/Crashes with fatality by driver type-1.png" style="display: block; margin: auto;" />
When we restrict only to those accidents with at least one fatality, as
in **Graph 12**, again in 51.54% of crashes there was at least one
aggressive driver, but then unbelted drivers come second. It means that
in 41.48% crashes that resulted in one or more deaths, there were
unbelted drivers. Wearing the belt while driving should also be a
priority in policies to diminish death on the road, as well as disuading
people from drink and driving, and speeding.

Please notice that the number of graphs 11 and 12 do not add up to 100
becuase there are accidents were the drivers had two or more of the
mentioned conditions.

### 4. Regression Analysis

In this last section of the analysis, a couple of correlation matrices
are presented. There are some interesting correlations, such as the
negative one between aggressive driving and distracted driving. The use
of cellphone is related positively with distracted driving, as would be
expected. And also tailgating and speeding have a positive relation with
aggressive driving, which is also very understandable.

``` r
model_X_1 <- c('SEVERITY', 'DRINKING_DRIVER', 'DISTRACTED', 'UNBELTED', 'AGGRESSIVE_DRIVING', 'CELL_PHONE', 'FATIGUE_ASLEEP','ILLEGAL_DRUG_RELATED', 'SPEEDING', 'TAILGATING', 'YOUNG_DRIVER')
# after selecting a subset for correlations, compute correlation
model_X1_cor <- subset(crashes_df, select = model_X_1 )
# show in a graph
corrplot(cor(model_X1_cor))
```

<img src="Project_draft_files/figure-gfm/Correlation 1-1.png" style="display: block; margin: auto;" />
The negative correlation between local road and speed limit was
expected: local roads tend to have lower speed limits.

``` r
model_X_2 <- c('SEVERITY', 'ILLUMINATION_DARK', 'CURVED_ROAD', 'INTERSECTION', 'SPEED_LIMIT', 'LOCAL_ROAD', 'URBAN_RURAL')
model_X2_cor <- subset(crashes_df, select = model_X_2 )
corrplot(cor(model_X2_cor))
```

<img src="Project_draft_files/figure-gfm/Correlation 2-1.png" style="display: block; margin: auto;" />

For the regression analysis, a multinomial logistic regression is run.
In order not to plug everything into the model, I selected the variables
that might be more powerful to explain the severity of crashes. Also,
some variables were not included for potential multicollinearity, as is
the case of the ROAD_CONDITION variable that should suffer from almost
perfect correlation with the variable WEATHER.

The target variable is Severity, and the results are presented below:

``` r
# specification of regression model
model_1 = polr(SEVERITY_L ~ DRINKING_DRIVER + DISTRACTED + UNBELTED + AGGRESSIVE_DRIVING + CELL_PHONE + FATIGUE_ASLEEP + ILLEGAL_DRUG_RELATED + SPEEDING + TAILGATING + UNBELTED + YOUNG_DRIVER + ILLUMINATION_DARK +  CURVED_ROAD + INTERSECTION + URBAN_RURAL + SPEED_LIMIT + LOCAL_ROAD + as.factor(DAY_OF_WEEK) + as.factor(WEATHER), data = crashes_df, Hess = TRUE)
summary(model_1)
```

    ## Call:
    ## polr(formula = SEVERITY_L ~ DRINKING_DRIVER + DISTRACTED + UNBELTED + 
    ##     AGGRESSIVE_DRIVING + CELL_PHONE + FATIGUE_ASLEEP + ILLEGAL_DRUG_RELATED + 
    ##     SPEEDING + TAILGATING + UNBELTED + YOUNG_DRIVER + ILLUMINATION_DARK + 
    ##     CURVED_ROAD + INTERSECTION + URBAN_RURAL + SPEED_LIMIT + 
    ##     LOCAL_ROAD + as.factor(DAY_OF_WEEK) + as.factor(WEATHER), 
    ##     data = crashes_df, Hess = TRUE)
    ## 
    ## Coefficients:
    ##                                         Value Std. Error  t value
    ## DRINKING_DRIVER                      0.341828  0.0172476  19.8189
    ## DISTRACTED                           0.011015  0.0148924   0.7396
    ## UNBELTED                             0.809432  0.0138603  58.3994
    ## AGGRESSIVE_DRIVING                   0.085535  0.0103656   8.2519
    ## CELL_PHONE                          -0.055659  0.0449885  -1.2372
    ## FATIGUE_ASLEEP                       0.236628  0.0381284   6.2061
    ## ILLEGAL_DRUG_RELATED                 0.505003  0.0451894  11.1753
    ## SPEEDING                             0.333568  0.0235093  14.1888
    ## TAILGATING                          -0.012354  0.0214712  -0.5754
    ## YOUNG_DRIVER                        -0.037573  0.0124282  -3.0232
    ## ILLUMINATION_DARK                   -0.335677  0.0105225 -31.9010
    ## CURVED_ROAD                         -0.086713  0.0124096  -6.9876
    ## INTERSECTION                         0.230165  0.0107818  21.3476
    ## URBAN_RURAL                          0.043112  0.0147363   2.9255
    ## SPEED_LIMIT                          0.001494  0.0005008   2.9840
    ## LOCAL_ROAD                          -0.133065  0.0124862 -10.6570
    ## as.factor(DAY_OF_WEEK)Monday         0.054718  0.0183688   2.9789
    ## as.factor(DAY_OF_WEEK)Tuesday        0.046759  0.0181487   2.5765
    ## as.factor(DAY_OF_WEEK)Wednesday      0.061897  0.0180434   3.4304
    ## as.factor(DAY_OF_WEEK)Thursday      -0.003704  0.0179888  -0.2059
    ## as.factor(DAY_OF_WEEK)Friday         0.009068  0.0175140   0.5177
    ## as.factor(DAY_OF_WEEK)Saturday      -0.006871  0.0180059  -0.3816
    ## as.factor(WEATHER)Blowing snow      -0.095257  0.0149509  -6.3713
    ## as.factor(WEATHER)Clear             -0.359077  0.0165231 -21.7318
    ## as.factor(WEATHER)Cloudy            -0.406717  0.0191819 -21.2031
    ## as.factor(WEATHER)Fog/Smoke         -0.416089  0.0998349  -4.1678
    ## as.factor(WEATHER)Freezing rain     -0.251138  0.1146883  -2.1897
    ## as.factor(WEATHER)Rain              -0.440517  0.0408261 -10.7901
    ## as.factor(WEATHER)Severe crosswinds -0.270384  0.0522977  -5.1701
    ## as.factor(WEATHER)Sleet / hail      -0.129211  0.1102773  -1.1717
    ## as.factor(WEATHER)Snow              -0.835996  0.0657105 -12.7224
    ## 
    ## Intercepts:
    ##                                                           Value    Std. Error
    ## (0) No Damages|(1) Only property damages                   -4.1062   0.0550  
    ## (1) Only property damages|(2) Only 1 injured                0.3833   0.0516  
    ## (2) Only 1 injured|(3) Several injured, no deaths           2.2740   0.0520  
    ## (3) Several injured, no deaths|(4) Fatalities (1 or many)   5.4756   0.0605  
    ##                                                           t value 
    ## (0) No Damages|(1) Only property damages                  -74.6023
    ## (1) Only property damages|(2) Only 1 injured                7.4261
    ## (2) Only 1 injured|(3) Several injured, no deaths          43.7545
    ## (3) Several injured, no deaths|(4) Fatalities (1 or many)  90.5645
    ## 
    ## Residual Deviance: 369003.10 
    ## AIC: 369073.10

I will not describe all the regression ouptut, but rather focus on the
most interesting results:

-   **Drinking driver.** It has a positive coefficient, meaning that
    drinking drivers are related to a higher probability of having a
    crash with more severe consequences. This makes sense, and we have
    been observing that in the data. One important recommendation for
    policy makers is to continue focusing on drink and driving.
-   **Unbelted.** Unbelted drivers are associated with higher severity
    of accidents. A policy that enforces wearing the seatbelt could also
    lead to positive results.
-   **Intersection.** Its coefficient has a potivie sign as well, so
    paying attention to interesections is important. Making sure that
    signs are visibile, and maybe event randomly patrolling the
    intersections where most accidents happen coul be an interesting
    policy.
-   **Local road.** Local roads has a negative coefficient, this might
    be related to speed limits.
-   **Snow.** Interestingly, snow has also a negative coefficient.
    Probably because people are bein more cuatios when driving during
    snow, and there fore accidents are less severe.

This was an initial regression analysis, and some improvements can be
made to have a more robust regression model. Nevertheless, the results
are interesting and could be of utility for orienting policy
recommendation.

### 5. Conclusions and future work

As a very brief conclusion, the data is very complete and interesting
and there are other variables that due to time restrictions were not
analysed thouroughly in this project, but that could serve for
additional research.

The results of the descriptive analysis and the regression model are
consistent, and give a pretty clear idea of the most important factors
that increase or decrease the severity of crashes in Allegheny County.

Some of the limitations of this analysis include: most variables are
catorgircal, so the diversity of graphs to present the data was a little
limited; also, it would have been interesting to have a time indicator
to perform time series analysis.

### References

I used some references, mainly as help for polishing my graphs.  
<https://dplyr.tidyverse.org/reference/summarise.html>  
<https://stackoverflow.com/questions/24027605/determine-the-number-of-na-values-in-a-column>  
Got some help for the use of strptime:  
<https://www.rdocumentation.org/packages/base/versions/3.6.2/topics/strptime>  
Help with a couple of graphs:  
<http://r-statistics.co/Top50-Ggplot2-Visualizations-MasterList-R-Code.html>  
Got some help for graphing with plotly, especially for changing the
layout of the graph  
<https://stackoverflow.com/questions/37809906/control-which-tick-marks-labels-appear-on-x-axis-in-plotly>  
To create a new column based on criteria of other columns:  
<https://stackoverflow.com/questions/12125980/how-to-create-a-new-variable-in-a-data-frame-based-on-a-condition>  
<https://www.marsja.se/r-add-column-to-dataframe-based-on-other-columns-conditions-dplyr/>

[1] The data set can be downloaded from:
<https://data.wprdc.org/dataset/allegheny-county-crash-data/resource/2c13021f-74a9-4289-a1e5-fe0472c89881>
