``` r
library(data.table)
library(tidyverse)
```

#### resources

free intro to data manipulation with `dplyr` and `data.table` through [datacamp](https://www.datacamp.com/home) (more than intro costs $)

-   [dplyr](https://www.datacamp.com/courses/dplyr-data-manipulation-r-tutorial)
-   [data.table](https://campus.datacamp.com/courses/data-table-data-manipulation-r-tutorial)

Creating/Inspecting dataframes
==============================

Creating a dataframe in `dplyr` or `data.table` is very similiar to creating a dataframe in base `R`. An advantage of dataframe creation for both packages is that `stringAsFactors` defaults to `FALSE`.

Examining the structure of both df/dt we can start to see some differences. The `dplyr` dataframe has classes `tbl_df`, `tbl`, & `data.frame`. The `data.table` has classes `data.table` & `data.frame`. They both have class `data.frame` so we'll be able to call functions written for the base `R` class.

The last difference we can see in the sturcture is that `data.table` has a pointer attribute named `.internal.selfref`. This attribute will allow us to modify the `data.table` by reference for some operations and avoid copy-on-modify.

Lastly, we can print the two dataframes and see that they both have console friendly print methods for large datasets.

``` r
len      <- 1e5
col_inds <- 1:len
col_grps <- sample(letters[1:3], len, replace=TRUE)
col_vals <- rnorm(len)

#df creation-----------------------------------------
df <- dplyr::data_frame(ind = col_inds,
                        grp = col_grps,
                        val = col_vals)

dt <- data.table::data.table(ind = col_inds,
                             grp = col_grps,
                             val = col_vals)
#df structure-----------------------------------------
str(df)
```

    ## Classes 'tbl_df', 'tbl' and 'data.frame':    100000 obs. of  3 variables:
    ##  $ ind: int  1 2 3 4 5 6 7 8 9 10 ...
    ##  $ grp: chr  "b" "a" "c" "b" ...
    ##  $ val: num  -0.5453 -1.4466 0.0808 -0.6143 -1.5132 ...

``` r
str(dt)
```

    ## Classes 'data.table' and 'data.frame':   100000 obs. of  3 variables:
    ##  $ ind: int  1 2 3 4 5 6 7 8 9 10 ...
    ##  $ grp: chr  "b" "a" "c" "b" ...
    ##  $ val: num  -0.5453 -1.4466 0.0808 -0.6143 -1.5132 ...
    ##  - attr(*, ".internal.selfref")=<externalptr>

``` r
#print methods----------------------------------------
df
```

    ## # A tibble: 100,000 × 3
    ##      ind   grp         val
    ##    <int> <chr>       <dbl>
    ## 1      1     b -0.54528051
    ## 2      2     a -1.44659698
    ## 3      3     c  0.08082766
    ## 4      4     b -0.61431850
    ## 5      5     c -1.51320799
    ## 6      6     b -0.32294980
    ## 7      7     c  0.40916513
    ## 8      8     c  0.74801291
    ## 9      9     b  0.62383372
    ## 10    10     a  0.19813085
    ## # ... with 99,990 more rows

``` r
dt
```

    ##            ind grp         val
    ##      1:      1   b -0.54528051
    ##      2:      2   a -1.44659698
    ##      3:      3   c  0.08082766
    ##      4:      4   b -0.61431850
    ##      5:      5   c -1.51320799
    ##     ---                       
    ##  99996:  99996   b  0.38383444
    ##  99997:  99997   c  0.63775732
    ##  99998:  99998   a  1.58754273
    ##  99999:  99999   c -1.76046752
    ## 100000: 100000   a -0.36833525

Writing/Reading csvs
====================

**note: these timings were performed on windows 64bit (8core, 32gb ram); `readr::write_csv` performs much better writing this 100000 row df on a non windows machine**

Writing
-------

Again, the syntax for writing csvs in both frameworks is very similiar to writing a csv in base. An advantage of both over base, in my opinion, is the omission of writing row.names by default. The `fwrite` function has the ability to set `row.names=TRUE` while the `readr` implementation does not have an argument for rownames.

`fwrite` stands for 'fast write' and it lives up to the hype. In our example 100,000 rows the `readr` implentation is decidely slower (16375% slower for this knit). From `?fwrite` documentation: 'Modern machines almost surely have more than one CPU so fwrite uses them.'

``` r
system.time(readr::write_csv(df, "readr_out.csv"))
```

    ##    user  system elapsed 
    ##    0.15    0.82    6.55

``` r
system.time(data.table::fwrite(dt, "fwrite_out.csv"))
```

    ##    user  system elapsed 
    ##    0.02    0.00    0.04

Reading
-------

Just for fun we'll switch read in the file that the opposing package wrote out (WoOoOoOoO!)

With the current size of our data the read times for `readr` and `data.table` are very similiar; both are typically executing in under a second on my machine. In both functions we can specify the column classes. Using `readr` we can specify the `col_types` using shorthand or full names.

Both functions, of course, read the csv into the two different structures that we saw above.

This example doesn't show it, but `fread` scales to much larger data sets better than `readr`. On a 40 million row x 3 column data set `readr` completed the read in a little over 2.5 minutes, while `fread` completed the job in 14 seconds.

``` r
system.time(readr::read_csv("fwrite_out.csv", col_types = "icn"))
```

    ##    user  system elapsed 
    ##    0.02    0.03    0.14

``` r
system.time(data.table::fread("readr_out.csv", colClasses = c("integer", "character", "numeric")))
```

    ##    user  system elapsed 
    ##    0.10    0.00    0.17

Data Manipulation
=================

This is mostly going to be a collection of example syntax for performing operations in the different frameworks. Commentary on the code chunks will be limited.

*(note: this doc is geared towards `dplyr` users that are less familiar with `data.table`.)*

### data.table syntax intro

Operations in `data.table` primarily use `[`. In base `[` typically is used for subsetting and given just 2 arguments when used with a dataframe: rows (`i`) and columns (`j`). When used with a `data.table` the brackets assume new functionality. The `[` still take `i` & `j` like arguments but a 3rd argument (`by`) is now assumed to be a grouping variable (`dt[i, j, by]`). Other differences between `data.table` and include: ability to reference column names without `df$` syntax increased computational functionality in the `j` argument.

Filtering
---------

``` r
#dplyr
df %>% 
  filter(grp == "a")

#data.table
dt[grp=="a",]

#base
df[df$grp=="a",]
```

Sorting
-------

Sort by `grp`.

``` r
#dplyr
df %>% 
  arrange(grp)

#data.table
dt[order(grp),]
#setkey(dt, grp)

#base
df[order(df$grp),]
```

Creating/Storing/Deleting a column
==================================

Create & store new column
-------------------------

Here we see a few new items in the `data.table` methodology.

In `dplyr` in order to create a new column and store the result we will have to create the column and then copy our data.frame to a new address for storage (this move can be seen by the change in `address(df)`). However, the operator `:=` from `data.table` creates the new column by reference so we do not copy our table to a new address; this can be a big performance boost for large datasets.

The next thing we see in `data.table` is the introduction of `.N`, which evaluates to `nrow(dt)` when called from `` `[.data.table` ``. In the `dplyr` pipeline we get the number of rows using the `.` sytnax associated with `` `%>%` ``.

``` r
#dplyr
address(df) #"00000000691018D0"
df <- df %>% 
  mutate(new_col = runif(nrow(.)))
address(df) #"000000002A3825A8"

#data.table
address(dt) #"00000000117125C0"
dt[,new_col := runif(.N)]
address(dt) #"00000000117125C0"
#dt[,.(ind, grp, val, new_col = runif(.N))]
```

Delete column
-------------

``` r
df <- df %>% 
  select(-new_col)

dt[,new_col := NULL]
```

Create column without storing
-----------------------------

Some new shorthand is again introced in the `data.table` syntax used here. The `.SD` references all columns not included in the grouping `by` argument of `` `[.data.table` ``. This evaluates to a list of columns (i.e. a dataframe), and we can add columns by `c`ombining a new list of columns to `.SD`. If `j` evaluates to a list of equal lenght columns `` `[.data.table` `` will interpret it as a `data.table`.

``` r
df %>% 
  mutate(new_col = runif(nrow(.)))
```

    ## # A tibble: 100,000 × 4
    ##      ind   grp         val   new_col
    ##    <int> <chr>       <dbl>     <dbl>
    ## 1      1     b -0.54528051 0.4302584
    ## 2      2     a -1.44659698 0.6446282
    ## 3      3     c  0.08082766 0.5114203
    ## 4      4     b -0.61431850 0.9204234
    ## 5      5     c -1.51320799 0.6493419
    ## 6      6     b -0.32294980 0.5688315
    ## 7      7     c  0.40916513 0.1372726
    ## 8      8     c  0.74801291 0.8770839
    ## 9      9     b  0.62383372 0.7419305
    ## 10    10     a  0.19813085 0.2833246
    ## # ... with 99,990 more rows

``` r
dt[,c(.SD, list(newnew = runif(.N)))]
```

    ##            ind grp         val     newnew
    ##      1:      1   b -0.54528051 0.54285282
    ##      2:      2   a -1.44659698 0.96687137
    ##      3:      3   c  0.08082766 0.94399398
    ##      4:      4   b -0.61431850 0.06784121
    ##      5:      5   c -1.51320799 0.99183871
    ##     ---                                  
    ##  99996:  99996   b  0.38383444 0.20356066
    ##  99997:  99997   c  0.63775732 0.22234155
    ##  99998:  99998   a  1.58754273 0.74215794
    ##  99999:  99999   c -1.76046752 0.13666365
    ## 100000: 100000   a -0.36833525 0.95060147

Summarising and Grouping
========================

*note: `dplyr` applies sorting by the grouping variable when summarising; `data.table` orders the summarization by the first appearance of each distinct value in the grouping variable*

``` r
#using default naming
df %>% 
  group_by(grp) %>% 
  summarise(mean(val))
```

    ## # A tibble: 3 × 2
    ##     grp   `mean(val)`
    ##   <chr>         <dbl>
    ## 1     a  0.0002478248
    ## 2     b -0.0086373767
    ## 3     c  0.0016920526

``` r
#using default naming
dt[,mean(val), by=grp]
```

    ##    grp            V1
    ## 1:   b -0.0086373767
    ## 2:   a  0.0002478248
    ## 3:   c  0.0016920526

``` r
#using custom naming
df %>% 
  group_by(grp) %>% 
  summarise(my_mean = mean(val))
```

    ## # A tibble: 3 × 2
    ##     grp       my_mean
    ##   <chr>         <dbl>
    ## 1     a  0.0002478248
    ## 2     b -0.0086373767
    ## 3     c  0.0016920526

``` r
#using custom naming
dt[,.(my_mean = mean(val)), by=grp]
```

    ##    grp       my_mean
    ## 1:   b -0.0086373767
    ## 2:   a  0.0002478248
    ## 3:   c  0.0016920526

Joins
=====

Left join
---------

`dplyr` has very descriptive `?join` functions, while `data.table` can use short hand of `dt[dt2]` or `merge`. The `merge` syntax is similiar the base `merge`.

``` r
df2 <- df
df %>% 
  left_join(df2, by="ind")

dt2 <- dt
setkey(dt, ind)
setkey(dt2, ind)

dt[dt2]
merge(dt, dt2, by = "ind", all.x = TRUE)
```

Transforming columns
====================

``` r
df %>% 
  mutate_all(as.character)
```

    ## # A tibble: 100,000 × 3
    ##      ind   grp                val
    ##    <chr> <chr>              <chr>
    ## 1      1     b -0.545280508170098
    ## 2      2     a  -1.44659697798738
    ## 3      3     c 0.0808276614999396
    ## 4      4     b -0.614318501663225
    ## 5      5     c  -1.51320799247204
    ## 6      6     b -0.322949797444465
    ## 7      7     c  0.409165129048615
    ## 8      8     c   0.74801290811267
    ## 9      9     b  0.623833723477989
    ## 10    10     a  0.198130852264119
    ## # ... with 99,990 more rows

``` r
dt[,lapply(.SD, as.character)]
```

    ##            ind grp                val
    ##      1:      1   b -0.545280508170098
    ##      2:      2   a  -1.44659697798738
    ##      3:      3   c 0.0808276614999396
    ##      4:      4   b -0.614318501663225
    ##      5:      5   c  -1.51320799247204
    ##     ---                              
    ##  99996:  99996   b  0.383834442934876
    ##  99997:  99997   c  0.637757323905886
    ##  99998:  99998   a   1.58754272774486
    ##  99999:  99999   c  -1.76046752347906
    ## 100000: 100000   a -0.368335246887932

Additional stuff
================

``` r
# count per group
df %>% count(grp)
```

    ## # A tibble: 3 × 2
    ##     grp     n
    ##   <chr> <int>
    ## 1     a 33570
    ## 2     b 33386
    ## 3     c 33044

``` r
dt[,.N, grp]
```

    ##    grp     N
    ## 1:   b 33386
    ## 2:   a 33570
    ## 3:   c 33044

``` r
#eval operation to vector
dt[,sum(val)]
```

    ## [1] -224.1358

``` r
#eval operation to vector where 
dt[grp=="b", sum(val)]
```

    ## [1] -288.3675

``` r
#referencing rows by key value
setkey(dt, grp)
dt[.("b"), sum(val)]
```

    ## [1] -288.3675

[last minute example thrown in](http://stackoverflow.com/questions/43957195/linear-interpolation-by-group-in-r/43957539#43957539)
