# dataQC

```R
# install.packages("devtools")
devtools::install_github("D-ESC/dataQC")
```

Some simple and hopefully robust ways to deal with detecting and possibly removing errors and inconsistencies from data.

```R
library(lubridate)
library(dplyr)

dates = as.POSIXct(seq(from = ymd("2013-01-01"), to = ymd("2014-01-01"), by = 3 * 3600))
Data = data.frame(dates, value = sin(decimal_date(dates)/0.01) + rnorm(length(dates)))
```

Interact with data by zooming in on certain regions of a scatterplot and highlighting values to be returned to the console. 

```R
library(dataQC)
Data = brushed(Data, "dates", "value")
```

This returns the original object with an additional column 'selected_' that can be used to flag or update values.

```R
Data = Data %>% 
  mutate(value = ifelse(selected_ == TRUE, NA, value)) %>%
  mutate(flag = ifelse(selected_ == TRUE, 'missing', flag))
```
## impossible values

Errors from a malfunctioning instrument can be filtered out.

```R
Data %>% 
  filter(value < 3 & value > -3)
```

## “potential outliers”

boxplot.stats can list the 'outliers' (points outside +/-1.58 IQR/sqrt(n)). Coefficient that defines the outliers can be changed.

```{r}
Data %>% 
  filter(value %in% boxplot.stats(value, coef = 1)$out)
```

adjboxStats computes the “statistics” for producing boxplots adjusted for skewed distributions. Scaling factors can be set to change outlier boundaries.

```R
library(robustbase)
Data %>% 
  filter(value %in% adjboxStats(value, a=-1,b=5)$out)
```

Any of the above can be calculated in groups such as year or month.

```R
Data %>% 
  group_by(month(dates)) %>% 
  filter(value %in% boxplot.stats(value)$out)
```

And we can set these values to NA.

```R
Data %>% 
  group_by(month(dates)) %>% 
  mutate(value = ifelse(value %in% boxplot.stats(value)$out, NA, value))
```

## NA Handling

Four methods for dealing with NAs (missing observations) can be found in the zoo package: na.contiguous, na.approx and na.locf. Also, na.omit returns an object with incomplete observations removed; na.contiguous extracts the longest consecutive stretch of non-missing values.

Missing values (NAs) can be replaced by linear interpolation via na.approx or cubic spline interpolation via na.spline, respectively. “maxgap” can be used to set the maximum number of consecutive NAs to fill for na.approx. Any longer gaps will be left unchanged.

Generic function for replacing each NA with the most recent non-NA prior to it.

```R
library(zoo)
Data %>% 
  mutate(value = na.locf(value, maxgap = 6))
```
Generic function for replacing each NA with aggregated values. This allows imputing by the overall mean, by monthly means, etc.

```R
Data %>% 
  mutate(value = na.aggregate(value, by = month(dates), FUN = mean, maxgap = 6))
```

Missing values (NAs) are replaced by linear interpolation via approx.

```R
Data %>% mutate(value = na.approx(value, maxgap = 6))
```
