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

## Check and re-format data

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

### Recategorise individual level variables

Ensure each dataset contains the same information.

In this example we need to categorise age in the ind dataset to match the constraint data (con_age).


```r
# categorise (bin) the age variable
brks <- c(0,49,120) # set break points
labs <- c("a0_49", "a50+") # create labels
ind$age <- cut(ind$age, breaks = brks, labels = labs) # overwrite the age variable with categorical age bands
ind
```

```
##   id   age sex
## 1  1  a50+   m
## 2  2  a50+   m
## 3  3 a0_49   m
## 4  4  a50+   f
## 5  5 a0_49   f
```

### Match individual and aggregate level data names


```r
levels(ind$age) # what are the levels in individual age variable?
```

```
## [1] "a0_49" "a50+"
```

```r
names(con_age) # what are the column names (= age levels) in the aggregate age constraint variable
```

```
## [1] "a0.49" "a.50."
```

```r
names(con_age) <- levels(ind$age) # rename aggregate age constraint variables
```

### Create constraint object

Now all constraint variable names match the individual data we combine them into a single object.


```r
cons <- cbind(con_age, con_sex) # column bind, cols are constraints - rows are zones

cons[1:2, ] # display constraints for first two zones (rows)
```

```
##   a0_49 a50+ m f
## 1     8    4 6 6
## 2     2    8 4 6
```
We now have two objects for the individual and aggregate datasets:

* Individual - with dimensions (5, 3) - 5 individuals, 3 variables
* Constraints - with dimensions (3, 4) - 3 zones, 4 variables (2 variables with 2categories each)

### Make individual and constraint objects comparable

In order to compare the two datasets we must 'flatten' the individual level data - increasing the width so each column becomes one category name (and thus matching format of constraints data).

**Note: great care should be taken to format columns in the same order as aggregate (constraints) data** - unexpected results may occur where there are errors here. Be warned!


```r
cat_age <- model.matrix(~ ind$age - 1)
cat_sex <- model.matrix(~ ind$sex - 1)[, c(2, 1)] # square brackets changes column order (to match constraints dataset)

(ind_cat <-  cbind(cat_age, cat_sex)) # combine into single data frame - brackets used to print result
```

```
##   ind$agea0_49 ind$agea50+ ind$sexm ind$sexf
## 1            0           1        1        0
## 2            0           1        1        0
## 3            1           0        1        0
## 4            0           1        0        1
## 5            1           0        0        1
```

```r
(colSums(ind_cat)) # view the aggregated version of ind (and print)
```

```
## ind$agea0_49  ind$agea50+     ind$sexm     ind$sexf 
##            2            3            3            2
```

```r
ind_agg <- colSums(ind_cat) # save result

# test compatability of ind_agg and cons 
(rbind(cons[1,], ind_agg)) # by binding into single data frame (uses only first row of cons)
```

```
##   a0_49 a50+ m f
## 1     8    4 6 6
## 2     2    3 3 2
```

## Population synthesis

Now that we have the data loaded and prepared this part is concerned with running a spatial microsimulation model using iterative proportional fitting (IPF). IPF is used to allocate individuals to zones.

> How representative each individual is of each zone is represented by their *weight* for that zone.
> The number of weights is therefore equal to the number of zones multiplied by the number of individuals in the microdata.

We have 3 rows and 5 individuals in the SimpleWorld data, therefore 15 weights will be estmiated in this example.

### Create a matrix to hold weights

First, we create a matrix and initially populate with 1s. The weights matrix links the individual data to he aggregate level. Therefore every individual in this table is currently equally representative of every zone.

A value of zero in cell `[i,j]` indicates that an individual `i` is not representative of a zone `j`.


```r
weights <- matrix(data = 1, nrow = nrow(ind), ncol = nrow(cons)) # create matrix for weights - set values as 1
(dim(weights))
```

```
## [1] 5 3
```

```r
weights
```

```
##      [,1] [,2] [,3]
## [1,]    1    1    1
## [2,]    1    1    1
## [3,]    1    1    1
## [4,]    1    1    1
## [5,]    1    1    1
```

During the IPF procedure, the weights in this matrix are iteratively updated until they converge towards a single result.

# Plots

You can also embed plots, for example:



Note that the `echo = FALSE` parameter was added to the code chunk to prevent printing of the R code that generated the plot.
