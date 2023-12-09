
<!-- README.md is generated from README.Rmd. Please edit that file -->

# queuebee <a href=''><img src='' align="right" height="200" /></a>

<!-- badges: start -->
<!-- badges: end -->

## Overview

## Installation

Install the official version from CRAN:

``` r
# install.packages("bitfield")
```

Install the latest development version from github:

``` r
devtools::install_github("luckinet/bitfield")
```

## Examples

``` r
library(bitfield)

library(dplyr, warn.conflicts = FALSE)
library(CoordinateCleaner)
library(stringr)
```

Let’s first build an example dataset

``` r
input <- tibble(x = sample(seq(23.3, 28.1, 0.1), 10),
                y = sample(seq(57.5, 59.6, 0.1), 10),
                year = rep(2021, 10),
                commodity = rep(c("soybean", "maize"), 5),
                some_other = rnorm(10))

# make it have some errors
input$x[5] <- 259
input$x[9] <- 0
input$y[10] <- NA_real_
input$y[9] <- 0
input$year[c(2:3)] <- c(NA, "2021r")
input$commodity[c(3, 5)] <- c(NA_character_, "dog")

# derive valid values for commodities
validComm <- c("soybean", "maize")
```

The first step in yielding quality bits is in creating a bitfield.

1.  The `width =` specifies how many bits are in the bitfield.
2.  The `lenght =` specifies how long the output table is. Any input
    must then have the same length. In case there are inputs with a
    different length, make sure in your pre-processing steps that all
    inputs are joined/merged that empty values or NAs are explicit.
3.  The `name =` specifies the label of the bitfield, which becomes very
    important when publishing, because bitfield and output table are
    stored in different files and it must be possible to unambiguously
    associate them to one another.

``` r
newBitfield <- qb_create(width = 10, length = dim(input)[1])
```

Then, individual bits need to be grown by specifying from which input
and based on which mapping function a particular position of the
bitfield should be modified. When providing a concise yet expressive
description, you will also in the future still understand what you did
here.

``` r
newBitfield <- newBitfield %>%
  # explicit tests for coordinates ...
  qb_grow(bit = qb_na(x = input, test = "x"), name = "is_na_x",
          desc = c("x-coordinate values do not contain any NAs"),
          pos = 1, bitfield = .) %>%
  qb_grow(bit =  qb_range(x = input, test = "x", min = -180, max = 180), name = "range_1_x",
          desc = c("x-coordinate values are numeric and within the valid WGS84 range"),
          pos = 2, bitfield = .) %>%
  # ... or override NA test
  qb_grow(bit = qb_range(x = input, test = "y", min = -90, max = 90), name = "range_1_y",
          desc = c("y-coordinate values are numeric and within the valid WGS84 range, NAs are FALSE"),
          pos = 3, na_val = FALSE, bitfield = .) %>%
  # it is also possible to use other functions that give flags, such as from CoordinateCleaner ...
  qb_grow(bit = cc_equ(x = input, lon = "x", lat = "y", value = "flagged"), name = "equal_coords",
          desc = c("x and y coordinates are not identical, NAs are FALSE"),
          pos = 4, na_val = FALSE, bitfield = .) %>%
  # ... or stringr ...
  qb_grow(bit = str_detect(input$year, "r"), name = "flag_year",
          desc = c("year values do have a flag, NAs are FALSE"),
          pos = 5, na_val = FALSE, bitfield = .) %>%
  # ... or even base R
  qb_grow(bit = !is.na(as.integer(input$year)), name = "is_na_year",
          desc = c("year values are valid integers"),
          pos = 6, bitfield = .)  %>%
  # test for matches with an external vector
  qb_grow(bit = qb_match(x = input, test = "commodity", against = validComm), name = "match_commodities",
          desc = c("commodity values are part of 'soybean' or 'maize'"),
          pos = 7, na_val = FALSE, bitfield = .) %>%
  # define cases
  qb_grow(bit = qb_case(x = input, some_other > 0.5, some_other > 0, some_other < 0, exclusive = FALSE), name = "cases_some_other",
          desc = c("some_other values are distinguished into large (. > 0.5), medium (. > 0) and small (. < 0)"),
          pos = 8:9, bitfield = .)
#> Testing equal lat/lon
#> Flagged NA records.
#> Warning in qb_grow(bit = !is.na(as.integer(input$year)), name = "is_na_year", :
#> NAs durch Umwandlung erzeugt
```

This strcuture is basically a record of all the things that are grown on
a bitfield, but so far nothing has happened

``` r
newBitfield
```

To make things happen eventually, the bitfield and a(ny) input it shall
be applied to are combined. This will result in an output table (with
one column that has the name `QB`) with the same length as the longest
column in the inputs. When bit flags are grown from inputs with
different length, a join/merge column needs to be provided.

``` r
(QB_int <- qb_combine(bitfield = newBitfield))
#> # A tibble: 10 × 1
#>       QB
#>    <int>
#>  1   495
#>  2   335
#>  3   415
#>  4   495
#>  5   429
#>  6   367
#>  7   495
#>  8   495
#>  9   487
#> 10   355
```

Anybody that wants to either extend the bitfield or analyse the output
data and base their judgement on the quality bit will want to extract
that bit. The bitfield is thereby the key that is required to decode the
quality bit

``` r
QB_chr <- qb_unpack(x = QB_int, bitfield = newBitfield, sep = "|") 
#> # A tibble: 8 × 4
#>   name              flags pos   description                                     
#>   <chr>             <int> <chr> <chr>                                           
#> 1 is_na_x               2 1     x-coordinate values do not contain any NAs      
#> 2 range_1_x             2 2     x-coordinate values are numeric and within the …
#> 3 range_1_y             2 3     y-coordinate values are numeric and within the …
#> 4 equal_coords          2 4     x and y coordinates are not identical, NAs are …
#> 5 flag_year             2 5     year values do have a flag, NAs are FALSE       
#> 6 is_na_year            2 6     year values are valid integers                  
#> 7 match_commodities     2 7     commodity values are part of 'soybean' or 'maiz…
#> 8 cases_some_other      3 8:9   some_other values are distinguished into large …

# prints legend by default, but it is also available in qb_env$legend

input %>% 
  bind_cols(QB_chr)
#> # A tibble: 10 × 7
#>        x     y year  commodity some_other    QB QB_chr          
#>    <dbl> <dbl> <chr> <chr>          <dbl> <int> <chr>           
#>  1  24.5  59.6 2021  soybean      -0.134    495 1|1|1|1|0|1|1|11
#>  2  24.7  57.6 <NA>  maize         1.17     335 1|1|1|1|0|0|1|01
#>  3  27.7  59   2021r <NA>         -0.559    415 1|1|1|1|1|0|0|11
#>  4  27.8  59.3 2021  maize        -2.39     495 1|1|1|1|0|1|1|11
#>  5 259    57.8 2021  dog          -3.93     429 1|0|1|1|0|1|0|11
#>  6  24.2  57.5 2021  maize         0.0905   367 1|1|1|1|0|1|1|01
#>  7  24.9  59.2 2021  soybean      -0.0455   495 1|1|1|1|0|1|1|11
#>  8  23.6  58.3 2021  maize        -2.68     495 1|1|1|1|0|1|1|11
#>  9   0     0   2021  soybean      -0.261    487 1|1|1|0|0|1|1|11
#> 10  23.9  NA   2021  maize         0.201    355 1|1|0|0|0|1|1|01
```

## Bitfields for other data-types

This example here shows how to compute quality bits for tabular data,
but this technique is especially helpful for raster data. Complex
workflows will, however, not only contain raster data, but a mix of
tabular and raster data, for example when occurrences and/or areal data
are modelled based on drivers that are available as rasters. In this
case the design of `queuebee` allows to use different of its’ functions
in different (distinct) parts of the workflow to build one overall QB.

To keep this package as simple as possible, no specific methods for
rasters were developed, rasters instead need to be converted to tabular
form and joined to the attributes or meta data that should be added to
the QB, for example like this

``` r
library(terra)

raster <- rast(matrix(data = 1:25, nrow = 5, ncol = 5))

input <- values(raster) %>% 
  as_tibble() %>% 
  rename(values = lyr.1) %>% 
  bind_cols(crds(raster), .)

# from here we can continue creating a bitfield and growing bits on it just like shown above...

# ... and then converting it back to a raster
QB_rast <- crds(raster) %>% 
  bind_cols(QB_int) %>% 
  rast(type="xyz", crs = crs(raster), extent = ext(raster))
```

## What is a bitfield, really?

In MODIS dataproducts a so-called “bit flag” (or [bit
field](https://en.wikipedia.org/wiki/Bit_field)) is used. This is a
combination of 0s and 1s that are combined into a binary sequence. It’s
called bit flag because each position in the sequence is coded by a
single bit. R stores each integer as a (huge) 32 bit value, you can
check this with `intToBits(x = 0L)` and see how many 0s are used up for
that (or actually 00s here, in hex-representation). This basically
means, that each integer in R can, by default, store 32 “switches” that
code for a particular information.

``` r
intToBits(x = 0L)
#>  [1] 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
#> [26] 00 00 00 00 00 00 00
intToBits(x = 1L)
#>  [1] 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
#> [26] 00 00 00 00 00 00 00

# or easier to read with .getBit
.getBit(2L)
#> [1] "01000000000000000000000000000000"
.getBit(3L, len = 2)
#> [1] "11"
```

Using the first two of those bits, we can represent 4 different states
of an attribute (00, 01, 10 and 11, or in the hex-notation that would be
`'00 00'`, `'00 01'`, `'01 00'` and `'01 01'`), with three bits 8 states
and with n bits 2^n states. This leaves lots of room to store all kind
of information in a single R integer.

By growing a bitfield with this package, you decide which information is
stored in which location, to represent not only data quality *per se*,
but all sort of information and perhaps even meta data.

When preparing a publication that contains (FAIR) data, it can therefore
be a good solution to provide not only the data table/layer, but also a
*single* additional column in a table or raster layer that records all
of those information, and the bitfield to decode the QB values.
