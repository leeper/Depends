This is a demo of the potential dangers of `Depends`. This repo contains three packages:

 - `dependsdplyr` Depends on dplyr and has no code
 - `dependsMASS` Depends on MASS and has no code
 - `dependsdplyr2` Depends on dplyr also, and uses the `select()` function without importing it or the dplyr namespace. Instead, it only `Depends` on dplyr.

This will highlight an important distinction between Depends - which *loads* a package namespace and *attaches* that namespace to the `search()` path - and Imports - which only loads the namespace.


Here's some setup using the installed packages:


```r
devtools::install("dependsdplyr", quick = TRUE, quiet = TRUE)
devtools::install("dependsdplyr2", quick = TRUE, quiet = TRUE)
devtools::install("dependsMASS", quick = TRUE, quiet = TRUE)
```

And here's a demo of the usual package load order problem with which we are all familiar:


```r
library("dependsdplyr")
```

```
## Loading required package: dplyr
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
head(select(mtcars, "cyl")) # works
```

```
##                   cyl
## Mazda RX4           6
## Mazda RX4 Wag       6
## Datsun 710          4
## Hornet 4 Drive      6
## Hornet Sportabout   8
## Valiant             6
```

```r
library("dependsMASS")
```

```
## Loading required package: MASS
```

```
## 
## Attaching package: 'MASS'
```

```
## The following object is masked from 'package:dplyr':
## 
##     select
```

```r
head(select(mtcars, "cyl")) # fails
```

```
## Error in select(mtcars, "cyl"): unused argument ("cyl")
```

```r
# remove packages from search() path
detach("package:dependsdplyr")
detach("package:dplyr")
detach("package:dependsMASS")
detach("package:MASS")
```

That shows that using Depends affects top-level code (i.e., user-created code in the R console). Not super surprising.

But here's the weird and possibly unexpected problem:


```r
library("dependsdplyr2")
```

```
## Loading required package: dplyr
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
"package:dplyr" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
"package:MASS" %in% search() # FALSE
```

```
## [1] FALSE
```

```r
# Here's a simple function for 'dependsdplyr2':
# dependsdplyr2::choose_cols() 
choose_cols
```

```
## function(x, cols) {
##     select(x, cols)
## }
## <bytecode: 0x0000000045039268>
## <environment: namespace:dependsdplyr2>
```

```r
# dependsdplyr2 function works as expected
head(choose_cols(mtcars, "cyl"))
```

```
##                   cyl
## Mazda RX4           6
## Mazda RX4 Wag       6
## Datsun 710          4
## Hornet 4 Drive      6
## Hornet Sportabout   8
## Valiant             6
```

```r
# now load MASS
library("MASS")
```

```
## 
## Attaching package: 'MASS'
```

```
## The following object is masked from 'package:dplyr':
## 
##     select
```

```r
"package:dplyr" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
"package:MASS" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
# dependsdplyr2 function errors
head(choose_cols(mtcars, "cyl"))
```

```
## Error in select(x, cols): unused argument (cols)
```

And the same error occurs if I attach MASS *indirectly* by loading a package that depends on it:


```r
detach("package:MASS")

# dependsdplyr2 function works as expected, again
head(choose_cols(mtcars, "cyl"))
```

```
##                   cyl
## Mazda RX4           6
## Mazda RX4 Wag       6
## Datsun 710          4
## Hornet 4 Drive      6
## Hornet Sportabout   8
## Valiant             6
```

```r
# now load dependsMASS
library("dependsMASS")
```

```
## Loading required package: MASS
```

```
## 
## Attaching package: 'MASS'
```

```
## The following object is masked from 'package:dplyr':
## 
##     select
```

```r
"package:dplyr" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
"package:MASS" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
# dependsdplyr2 function errors, again, even though I didn't explicitly attach MASS
head(choose_cols(mtcars, "cyl"))
```

```
## Error in select(x, cols): unused argument (cols)
```


This fails even though it's package code. Why? Because dependsdplyr2 is counting on dplyr being not only attached but also attached after any other package that might create namespace conflicts.

The lessons learned:

 - Always use a NAMESPACE to specify imports so that your package code isn't harmed by other peoples' use of Depends
 - Use Imports to specify any package that must be installed and *loaded* for your package to work
 - Always use a fully qualified reference - `pkg::func()` - when there might be some namespace ambiguity, such as at the top-level or when your package imports from two namespaces that conflict (like MASS and dplyr)
 - Don't use Depends in your packages

There are exceptions to the final rule (such as needing the methods package), but you never know what your use of Depends might do to someone else's top-level or package code.


---

Some cleanup:


```r
# detach
detach("package:dependsdplyr2")
detach("package:dependsMASS")

# remove packages
remove.packages(c("dependsdplyr", "dependsdplyr2", "dependsMASS"))
```

```
## Removing packages from 'C:/Program Files/R/R-3.5.0/library'
## (as 'lib' is unspecified)
```
