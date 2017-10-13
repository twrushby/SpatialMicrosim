# Spatial Microsim example
Thomas W Rushby  
13/10/2017  



# About

This document was created to record the code and output of the example spatial microsimulation in Robin Lovelace's book Spatial microsimulation with R <http://robinlovelace.net/spatial-microsim-book/>.

This is an R Markdown document. Markdown is a simple formatting syntax for authoring HTML, PDF, and MS Word documents. For more details on using R Markdown see <http://rmarkdown.rstudio.com>.

# Introduction

## Why use spatial microsim?

"Spatial Microsimulation is useful when you have an intermediary amount of data available: geographically aggregated count and a non-spatial survey." Lovelace - Spatial microsimulaton with R

\newpage

# Microsim example

## Load data


```r
### Load the individual level data

ind <- read.csv("Data/SimpleWorld/ind-full.csv") # load the individual level data

class(ind) # verify the data type
```

```
## [1] "data.frame"
```

```r
ind
```

```
##   id age sex income
## 1  1  59   m   2868
## 2  2  54   m   2474
## 3  3  35   m   2231
## 4  4  73   f   3152
## 5  5  49   f   2473
```

We can see from the output above that this has loaded a data frame object with 5 rows and 4 columns.


```r
### Load the constraint data (usually one variable at a time - individual files)

con_age <- read.csv("Data/SimpleWorld/age.csv")
con_sex <- read.csv("Data/SimpleWorld/sex.csv")

class(con_age)
```

```
## [1] "data.frame"
```

```r
class(con_sex)
```

```
## [1] "data.frame"
```

```r
con_age
```

```
##   a0.49 a.50.
## 1     8     4
## 2     2     8
## 3     7     4
```

```r
con_sex
```

```
##   m f
## 1 6 6
## 2 4 6
## 3 3 8
```

### Tests for constraint variables

Beware of constraint variables that come from different sources (Lovelace):
* check coherence of data
* in some cases the total number of individuals will be inconsistent
* can happen if constraint variables are measured using different base populations or at different levels (i.e indivdual and household)
This can cause the procedure to fail.

So, test constraints . . .

1. Zone ordering
Check that zones are loaded in the same order - and with some kind of zone_id that identifies each zone (consistent across datasets/constraint variables)
2. Total population
Indicates the same population base
3. Row totals
Indicating that the zones are listed in the same order (check on zone_ids) and the same population base


```r
sum(con_age) == sum(con_sex) # check population totals
```

```
## [1] TRUE
```

```r
rowSums(con_age) == rowSums(con_sex) # check row totals
```

```
## [1] TRUE TRUE TRUE
```

### Subset and filter

Good practice to filter out all unwanted data early on (i.e. remove superfluous variables). This will speed-up simulation and avoid over-complication.

In the ind dataset, only age and sex variables are useful so we remove income:


```r
ind_orig <- ind # keep original dataset
ind <- ind[, -4] # removes income (column 4)
ind
```

```
##   id age sex
## 1  1  59   m
## 2  2  54   m
## 3  3  35   m
## 4  4  73   f
## 5  5  49   f
```



# Plots

You can also embed plots, for example:



Note that the `echo = FALSE` parameter was added to the code chunk to prevent printing of the R code that generated the plot.
