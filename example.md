# Spatial Microsim example
Tom Rushby (t.w.rushby@soton.ac.uk) `@tom_rushby`  
Last run on: `r format(Sys.Date(), "%a %d %b %Y")`  



# About

This document was created to record the code and output of the example spatial microsimulation in Robin Lovelace's book Spatial microsimulation with R <http://robinlovelace.net/spatial-microsim-book/>. The text quotes heavily from the book and care should be taken when using any of the text to correctly cite the source.

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
# take each variable in turn and create a new matrix for each
# model.matrix expands factors of each variable, creating a column for each dummy variable (see help for more)
cat_age <- model.matrix(~ ind$age - 1)
cat_sex <- model.matrix(~ ind$sex - 1)[, c(2, 1)] # square brackets changes column order (to match constraints dataset)

(ind_cat <-  cbind(cat_age, cat_sex)) # combine matrices into single data frame - brackets used to print result
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

**note: creating matrix to hold weights appears here in book (printed pdf) but logically comes later - see weighting algorithms\ipf below**

## Weighting algorithms

There are *deterministic* and *stochastic* methods for weighting in spatial microsimulation. IPF is *deterministic* and therefore the results never vary: the weights will be the same every time. In contrast, *stochastic* methods use random numbers.

The distinction between *deterministic* and *stochastic* approaches points to a wider divide in methods: *reweighting* and *combinatorial optimisation*.

> The conecpt of weights is critical to understanding how population synthesis generates spatial microdata.

### Random allocation

If we have no information on the characteristics of the inhabitants, only total population of each zone then we can only assume that the distribution of characteristics found in the sample is representative of the distribution of the whole population. In this scenario, individuals are chosen at random from the sample and allocated to zones at random. Here, the distribution of characteristics of individuals in each zone will tend towards the microdata (see Lovelace, p.41).

In SimpleWorld this can be achieved by randomly allocating the 5 individuals of the microdata to zone 1 (with population of 12) using the `sample()` command:


```r
set.seed(1) # set seed for reproducibility
sel <- sample(x = 5, size = 12, replace = T) # create selection - sample uses a randmo number generator
ind_z1 <- ind_orig[sel, ]
head(ind_z1,3 )
```

```
##     id age sex income
## 2    2  54   m   2474
## 2.1  2  54   m   2474
## 3    3  35   m   2231
```

Changing the seed in the code above changes the individuals that are selected (try it).

### IPF

IPF is the most widely used and mature *deterministic* method to allocate individuals to zones (p.43).

> IPF invoves calculating a series of non-integer weights that represent how representative each individual is of each zone. This is reweighting. 

Three steps:

1. generate a weight matrix containing fractional numbers
2. integerisation - number of times each individual needs to be replicated (grossing up?)
3. expansion - calculation of the final dataset

#### Create a matrix to hold weights

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

#### IPFinR

IPFinR begins with a couple of nested loops, one to iterate through each zone (1:n_zone) and one to iterate through each category within the contraints (0-49 and 50+ in the first constraint, age).


```r
# create intuitive names for totals
n_zone <- nrow(cons) # number of zones
n_ind <- nrow(ind) # number of individuals
n_age <- ncol(con_age) #  number of categories of "age"
n_sex <- ncol(con_sex) # number of categories of "sex"

# create initial matrix of categorical counts from ind
# rows are zones and columns different categories of the variables
ind_agg0 <- t(apply(cons, 1, function(x) 1 *ind_agg))
colnames(ind_agg0) <- names(cons)
# duplicate the weight matrix to keep in memory each step
weights1 <- weights2 <- weights # create additional weight objects - all still populated with 1s at this stage

# Assign values to the previously created weight matrix
# to adapt to age constraint
for (j in 1:n_zone){
  for (i in 1:n_age){
    index <- ind_cat[, i] == 1
    weights1[index, j] <- weights[index, j] * con_age[j,i] / ind_agg0[j,i]
  }
  print(weights1)
}
```

```
##          [,1] [,2] [,3]
## [1,] 1.333333    1    1
## [2,] 1.333333    1    1
## [3,] 4.000000    1    1
## [4,] 1.333333    1    1
## [5,] 4.000000    1    1
##          [,1]     [,2] [,3]
## [1,] 1.333333 2.666667    1
## [2,] 1.333333 2.666667    1
## [3,] 4.000000 1.000000    1
## [4,] 1.333333 2.666667    1
## [5,] 4.000000 1.000000    1
##          [,1]     [,2]     [,3]
## [1,] 1.333333 2.666667 1.333333
## [2,] 1.333333 2.666667 1.333333
## [3,] 4.000000 1.000000 3.500000
## [4,] 1.333333 2.666667 1.333333
## [5,] 4.000000 1.000000 3.500000
```

To see weights that have been allocated to individuals to populate zone 2 we query the second column, giving the result:

2.6666667, 2.6666667, 1, 2.6666667, 1

To see the weight allocated to individual 3 for each zone we query the third row of the weight matrix:

4, 1, 3.5

Note: we ask R to write the result after each completing each zone. The algorithm proceeds zone by zone with each column of the matrix corresponding to a zone.
Note also that weights generated are fractional.

The next step is to re-aggregate the results from individual level after reweighting.
The weights of zone 1 (1st column of `weights1`) is multiplied by the characteristics of each individual (held in `ind_cat`). The result is a vector - the values corresponding to the number of people in each category for zone 1. To aggregate all individuals for zone 1, we sum the values in each category.

The following loop re-aggregates the individual level data with the new weights for each zone:


```r
ind_agg2 <- ind_agg1 <- ind_agg0 * NA # create additional ind_agg objects

# assign values to the aggregated data after constraint 1 - age
for(i in 1:n_zone){
  ind_agg1[i,] <- colSums(ind_cat * weights1[,i])
}

print(ind_agg1)
```

```
##      a0_49 a50+        m        f
## [1,]     8    4 6.666667 5.333333
## [2,]     2    8 6.333333 3.666667
## [3,]     7    4 6.166667 4.833333
```

#### Preliminary checks to ensure code is working correctly.

1. Are the resulting populations for each zone correct?


```r
# Check populations for first constraint variable - age (columns 1:2)
rowSums(ind_agg1[,1:2]) # simulated populations in each zone - age constraint only
```

```
## [1] 12 10 11
```

```r
rowSums(cons[,1:2]) # the observed populations in each zone
```

```
## [1] 12 10 11
```

```r
# Check populations for second constraint - sex (columns 3:4)
rowSums(ind_agg1[,3:4]) # simulated populations in each zone - sex constraint
```

```
## [1] 12 10 11
```

```r
rowSums(cons[,3:4]) # the observed populations in each zone
```

```
## [1] 12 10 11
```
2. What is the fit between observed and simulated results after constraining by age?

We calculate the correlation between aggregate actual data and the constraints. Produces a value between -1 and 1.
A value (in this example) of 1 will be perfect correlation.


```r
# test fit using cor function
# a 1d representation of the aggregate level data

vec <- function(x) as.numeric(as.matrix(x))
cor(vec(ind_agg0), vec(cons)) # before reweighting (ind_agg0)
```

```
## [1] -0.3368608
```

```r
cor(vec(ind_agg1), vec(cons)) # after reweighting using age constraint (ind_agg1)
```

```
## [1] 0.628434
```
We can see from the results above that the new weights lead to a much better fit.

#### Add second constraint variable


```r
for(j in 1:n_zone){
  for(i in 1:n_sex + n_age){
    index <- ind_cat[,i] == 1
    weights2[index,j] <- weights1[index,j] * cons[j,i] / ind_agg1[j,i]
  }
  print(weights2)
}
```

```
##      [,1] [,2] [,3]
## [1,]  1.2    1    1
## [2,]  1.2    1    1
## [3,]  3.6    1    1
## [4,]  1.5    1    1
## [5,]  4.5    1    1
##      [,1]      [,2] [,3]
## [1,]  1.2 1.6842105    1
## [2,]  1.2 1.6842105    1
## [3,]  3.6 0.6315789    1
## [4,]  1.5 4.3636364    1
## [5,]  4.5 1.6363636    1
##      [,1]      [,2]      [,3]
## [1,]  1.2 1.6842105 0.6486486
## [2,]  1.2 1.6842105 0.6486486
## [3,]  3.6 0.6315789 1.7027027
## [4,]  1.5 4.3636364 2.2068966
## [5,]  4.5 1.6363636 5.7931034
```

```r
# re-aggregate the individual level data with the new weights for each zone
# assign values to the aggregated data after constraint 2 - sex
for(i in 1:n_zone){
  ind_agg2[i,] <- colSums(ind_cat * weights2[,i])
}

print(ind_agg2)
```

```
##         a0_49     a50+ m f
## [1,] 8.100000 3.900000 6 6
## [2,] 2.267943 7.732057 4 6
## [3,] 7.495806 3.504194 3 8
```

```r
# as before, test fit using cor function

cor(vec(ind_agg0), vec(cons)) # before reweighting (ind_agg0)
```

```
## [1] -0.3368608
```

```r
cor(vec(ind_agg1), vec(cons)) # after reweighting using age constraint (ind_agg1) - first iteration
```

```
## [1] 0.628434
```

```r
cor(vec(ind_agg2), vec(cons)) # after reweighting using sex constraint (ind_agg2) - first iteration
```

```
## [1] 0.9931992
```

### IPF with ipfp


```r
# install.packages("ipfp")
library(ipfp)

# convert input constraint dataset to numeric - ipfp requires numeric data
cons <- apply(cons, 2, as.numeric) # to 1d numeric data type

# run ipfp for one zone
ipfp(cons[1,], t(ind_cat), x0 = rep(1, n_ind)) # runs ipf command
```

```
## [1] 1.227998 1.227998 3.544004 1.544004 4.455996
```

```r
# to avoid transposing ind_cat each time we call ipfp we create a new data frame
ind_catt <- t(ind_cat) # save transposed version of ind_cat
# create x0 to save creating object each time
x0 <- rep(1, n_ind) # an initial vector of 1s - one for each individual - the starting point of the weight estimates in ipfp

# simplified call for ipfp - 1 zone
ipfp(cons[1,], ind_catt, x0, v = TRUE) # runs ipf command - v = TRUE gives feedback
```

```
## iteration 0:	0.141421
## iteration 1:	0.00367328
## iteration 2:	9.54727e-05
## iteration 3:	2.48149e-06
## iteration 4:	6.44977e-08
## iteration 5:	1.6764e-09
```

```
## [1] 1.227998 1.227998 3.544004 1.544004 4.455996
```

```r
# loop to iterate through constraints, one zone (row) at a time
weights_loop_1 <- weights # first create a copy of the weights object

for(i in 1:ncol(weights)){
  weights_loop_1[,i] <- ipfp(cons[i,],ind_catt, x0, maxit = 2)
}

print(weights_loop_1)
```

```
##          [,1]      [,2]      [,3]
## [1,] 1.227273 1.7244202 0.7233239
## [2,] 1.227273 1.7244202 0.7233239
## [3,] 3.545455 0.5511596 1.5533522
## [4,] 1.542857 4.5467626 2.5416839
## [5,] 4.457143 1.4532374 5.4583161
```

```r
# alternatively we can use apply to loop over zones
weights_loop_2 <- weights # first create another copy of the weights object
# apply ipfp over rows in cons (set MARGIN = 1)
weights_loop_2 <- apply(cons, MARGIN = 1, FUN = 
                           function(x) ipfp(x, ind_catt, x0, maxit = 20))

print(weights_loop_2)
```

```
##          [,1]      [,2]      [,3]
## [1,] 1.227998 1.7250828 0.7250828
## [2,] 1.227998 1.7250828 0.7250828
## [3,] 3.544004 0.5498344 1.5498344
## [4,] 1.544004 4.5498344 2.5498344
## [5,] 4.455996 1.4501656 5.4501656
```
Don't forget to check the resulting weights obtained thru ipfp


```r
# Loop 1 aggregated individual weights - ipfp results, 2 iterations
ind_agg_loop1 <- t(apply(weights_loop_1, 2, function(x) colSums(x * ind_cat)))
colnames(ind_agg_loop1) <- colnames(cons)
print(ind_agg_loop1)
```

```
##         a0_49     a50+ m f
## [1,] 8.002597 3.997403 6 6
## [2,] 2.004397 7.995603 4 6
## [3,] 7.011668 3.988332 3 8
```

```r
# Loop 2 aggregated individual weights - ipfp results, 20 iterations
ind_agg_loop2 <- t(apply(weights_loop_2, 2, function(x) colSums(x * ind_cat)))
colnames(ind_agg_loop2) <- colnames(cons)
print(ind_agg_loop2)
```

```
##      a0_49 a50+ m f
## [1,]     8    4 6 6
## [2,]     2    8 4 6
## [3,]     7    4 3 8
```

```r
# Compare with constaints (aggregate) data
print(cons)
```

```
##      a0_49 a50+ m f
## [1,]     8    4 6 6
## [2,]     2    8 4 6
## [3,]     7    4 3 8
```

