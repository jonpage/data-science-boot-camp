Section 3: Anomaly Detection
================

-   [Introduction](#introduction)
-   [Problem](#problem)
-   [Data](#data)
    -   [Data Summary](#data-summary)
    -   [Training Set](#training-set)
-   [Modeling](#modeling)
    -   [Naive Model](#naive-model)
-   [Twitter's Anomaly Detection Library](#twitters-anomaly-detection-library)
-   [Anomalous](#anomalous)
-   [tsoutliers](#tsoutliers)

Introduction
------------

Anomalies are deviations from the norm or our expectations.

For modeling a time-series we need to find a stationary representation. If we have a non-stationary series (e.g., always growing), each new value may be outside of the bounds of the past observations. In this case we cannot build an expectation as a summary of past observations without including the growth rate. The first difference of such a series may be stationary, in which case we can model the fluctuations about its growth path.

The rest of this discussion focuses on the case of stationary series.

The general goal is

1.  Form a prediction/expected range
2.  Flag individuals that are outside of the expected range

Problem
-------

The problem we will look at in this section is identifying anomalies in time series. Time series can be represented in the time dimension, the frequency dimension, or via delay embeddings. We'll focus on the time dimension for this section.

Data
----

The date we'll use is the [NYC vehicle collision data set available on Kaggle](https://www.kaggle.com/nypd/vehicle-collisions).

``` r
rm(list=ls())
library(tidyverse)
```

    ## Loading tidyverse: ggplot2
    ## Loading tidyverse: tibble
    ## Loading tidyverse: tidyr
    ## Loading tidyverse: readr
    ## Loading tidyverse: purrr
    ## Loading tidyverse: dplyr

    ## Conflicts with tidy packages ----------------------------------------------

    ## filter(): dplyr, stats
    ## lag():    dplyr, stats

``` r
collisions <- read_csv('data/vehicle-collisions.csv')
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_character(),
    ##   `UNIQUE KEY` = col_integer(),
    ##   TIME = col_time(format = ""),
    ##   `ZIP CODE` = col_integer(),
    ##   LATITUDE = col_double(),
    ##   LONGITUDE = col_double(),
    ##   `PERSONS INJURED` = col_integer(),
    ##   `PERSONS KILLED` = col_integer(),
    ##   `PEDESTRIANS INJURED` = col_integer(),
    ##   `PEDESTRIANS KILLED` = col_integer(),
    ##   `CYCLISTS INJURED` = col_integer(),
    ##   `CYCLISTS KILLED` = col_integer(),
    ##   `MOTORISTS INJURED` = col_integer(),
    ##   `MOTORISTS KILLED` = col_integer()
    ## )

    ## See spec(...) for full column specifications.

### Data Summary

Let's see the bounds of the time series and pick a range for our test set.

``` r
library(lubridate)
```

    ## 
    ## Attaching package: 'lubridate'

    ## The following object is masked from 'package:base':
    ## 
    ##     date

``` r
collisions <- collisions %>%
  mutate(
    date = mdy(DATE),
    day_of_week = wday(date, label = TRUE),
    day_of_month = mday(date)
    )
collisions %>%
  summarise(start = min(date), end = max(date))
```

    ## # A tibble: 1 x 2
    ##        start        end
    ##       <date>     <date>
    ## 1 2015-01-01 2017-02-28

For this section, we will use 2015 for our training set. Having a full year of data is useful for finding normal bounds within the seasonality, which is most likely present in this data.

### Training Set

Let's create our `training` set and check the seasonality assumption.

``` r
training <- collisions %>%
  filter(date < ymd("2016-01-01"))
training_months <- training %>%
  mutate(month = as.factor(month(date, label = TRUE))) %>%
  group_by(month) %>%
  summarize(
    fatalities = sum(`PERSONS KILLED`) / days_in_month(first(date)),
    collisions = n() / days_in_month(first(date)),
    death_rate = fatalities / collisions)

training_months %>%
  ggplot(aes(month, collisions)) + geom_col() + ylim(0,NA) + 
  ggtitle("Average number of collisions per day")
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-3-1.png)

What about the number of fatalities?

``` r
ggplot(training_months, aes(month, fatalities)) + geom_col() + ylim(0,NA) + 
  ggtitle("Average number of fatalities per day")
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-4-1.png)

Does there appear to be seasonality in the rate of fatalities per collision?

``` r
ggplot(training_months, aes(month, death_rate)) + geom_col() + ylim(0,NA) + 
  ggtitle("Death by collision rate")
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-5-1.png)

To accurately assess the level of seasonality, it would be best for us to have multiple years. To avoid overfitting, we will live within the limits of only using one year of training data. It is clear from even this small sample that had we only selected January through March we would underestimate the number of traffic fatalities for all but September.

Let's see if there appears to be a pattern with respect to days of the week and days of the month.

``` r
training_days <- training %>%
  group_by(date) %>%
  summarize(
    fatalities = sum(`PERSONS KILLED`),
    day_of_week = first(day_of_week)
    )
ggplot(training_days, aes(day_of_week, fatalities)) + geom_violin()
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-6-1.png)

Total deaths by day of the week:

``` r
ggplot(training_days, aes(day_of_week, fatalities)) + geom_col()
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-7-1.png)

And now fatalities by day of the month:

``` r
training_dates <- training %>%
  group_by(day_of_month) %>%
  summarize(fatalities = sum(`PERSONS KILLED`))

ggplot(training_dates, 
       aes(((day_of_month - 1) %% 7) + 1, ((day_of_month - 1) %/% 7) + 1)) +
  geom_raster(aes(fill = fatalities)) + ylim(5,0) +
  ylab("") + xlab("") + ggtitle("Calendar Heatmap")
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-8-1.png)

Modeling
--------

Let's start by predicting the number of collisions per day.

``` r
training_series <- training %>%
  group_by(date) %>%
  summarize(
    collisions = n(), 
    injuries = sum(`PERSONS INJURED`), 
    fatalities = sum(`PERSONS KILLED`)
    )

ggplot(training_series, aes(date, collisions)) + geom_line()
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-9-1.png)

### Naive Model

A simple approach is to assume a roughly Gaussian process and to use a rule that we want to flag observations that are very unlikely.

Let's see how Gaussian this data is:

``` r
ggplot(training_series, aes(collisions)) + geom_density() +
  stat_function(fun=dnorm, color = "red", args = list(mean = mean(training_series$collisions), sd = sd(training_series$collisions)))
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-1.png)

For rarer events, we'll predict the number of injuries (or fatalities).

A simple anomaly detector can flag days where the observations are greater than x standard deviations from the mean.

``` r
mean_collisions = mean(training_series$collisions)
sd_collisions = sd(training_series$collisions)
moderate_collisions = mean_collisions + sd_collisions
high_collisions = mean_collisions + 2 * sd_collisions
extreme_collisions = mean_collisions + 3 * sd_collisions

ggplot(training_series, aes(date, collisions)) + 
  geom_hline(yintercept = moderate_collisions, color = "yellow") +
  geom_hline(yintercept = high_collisions, color = "orange") +
  geom_hline(yintercept = extreme_collisions, color = "red") +
  geom_line() +
  geom_point(data = training_series[training_series$collisions > extreme_collisions,], color = "red")
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-11-1.png)

Twitter's Anomaly Detection Library
-----------------------------------

See a blog post of this [here](https://blog.twitter.com/engineering/en_us/a/2015/introducing-practical-and-robust-anomaly-detection-in-a-time-series.html) and the paper describing their algorithm [here](https://www.usenix.org/system/files/conference/hotcloud14/hotcloud14-vallis.pdf). A common approach to time series modeling is Seasonal and Trend decomposition with LOESS ([Chapter on this topic](https://www.otexts.org/fpp/6/5)). The S-H-ESD algorithm in Twitter's library uses a piece-wise median

Install using the following commands

    install.packages("devtools")
    devtools::install_github("twitter/AnomalyDetection")

``` r
library(AnomalyDetection)
ts_result = training_series %>%
  mutate(datetime = as_datetime(date)) %>%
  select(datetime, collisions) %>%
  AnomalyDetectionTs(max_anoms=0.02, longterm = TRUE, piecewise_median_period_weeks = 8, direction='pos', plot=TRUE)
ts_result$plot
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-12-1.png)

``` r
cbind(date = as.character(ts_result$anoms$timestamp), anoms = ts_result$anoms$anoms)
```

    ##      date         anoms
    ## [1,] "2015-01-18" "960"
    ## [2,] "2015-03-06" "936"
    ## [3,] "2015-09-08" "795"

Anomalous
---------

``` r
#devtools::install_github("robjhyndman/anomalous")
library(anomalous)
```

    ## Loading required package: ForeCA

    ## Loading required package: ifultools

    ## This is 'ForeCA' version 0.2.4. Please see the NEWS file and citation("ForeCA").
    ## May the ForeC be with you.

    ## This is anomalous 0.1.0

``` r
y <- tsmeasures(training_series %>% select(collisions, injuries, fatalities))
y
```

    ##      lumpiness   entropy       ACF1    lshift   vchange cpoints fspots
    ## [1,] 0.4118487 0.9228047 0.35758024 0.9025886 0.8055127     136      5
    ## [2,] 0.1780018 0.8737697 0.44110942 0.8289241 0.4820937      93      5
    ## [3,] 0.4885690 0.9888523 0.03588016 0.8362327 0.8220356      28      7
    ##      trend linearity     curvature    spikiness  KLscore change.idx
    ## [1,]     0  4.340616 -2.628191e+00 2.195197e-05 4.888561         65
    ## [2,]     0  8.162027 -6.203614e+00 1.198940e-05 6.310924         18
    ## [3,]     0  1.285184 -2.183982e-09 3.119997e-05 9.645719         95
    ## attr(,"class")
    ## [1] "features" "matrix"

``` r
biplot(y)
```

    ## Warning in arrows(0, 0, y[, 1L] * 0.8, y[, 2L] * 0.8, col = col[2L], length
    ## = arrow.len): zero-length arrow is of indeterminate angle and so skipped

    ## Warning in arrows(0, 0, y[, 1L] * 0.8, y[, 2L] * 0.8, col = col[2L], length
    ## = arrow.len): zero-length arrow is of indeterminate angle and so skipped

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-14-1.png)

tsoutliers
----------

`tsoutliers` is another library providing outlier detection based on the methods of Chen and Liu (1993)

``` r
#install.packages("tsoutliers")
library(tsoutliers)

collisions_ts <- ts(training_series$collisions, c(2015, 1), frequency = 365)

collisions_outliers <- tso(y = collisions_ts)
collisions_outliers
```

    ## Series: collisions_ts 
    ## Regression with ARIMA(1,1,3)             errors 
    ## 
    ## Coefficients:
    ##          ar1      ma1      ma2     ma3      AO18       AO27
    ##       0.4562  -1.0063  -0.2161  0.2451  452.8651  -318.4295
    ## s.e.  0.4245   0.4183   0.2394  0.1760   78.1313    78.2153
    ## 
    ## sigma^2 estimated as 7865:  log likelihood=-2147.27
    ## AIC=4308.53   AICc=4308.84   BIC=4335.81
    ## 
    ## Outliers:
    ##   type ind    time coefhat  tstat
    ## 1   AO  18 2015:18   452.9  5.796
    ## 2   AO  27 2015:27  -318.4 -4.071

Type `AO` stands for Additive Outlier.

`tsoutliers` also provides a handy visualization of the identified outlier.

``` r
plot(collisions_outliers)
```

![](section-3-anomaly-detection_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-16-1.png)
