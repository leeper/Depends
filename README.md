This is a demo of the potential dangers of `Depends`. As [Hadley said on Twitter](https://twitter.com/hadleywickham/status/1003986395344470016), " it takes a while to fully grasp that DESCRIPTION primary influences the installation of your package, not is behaviour at run time. Depends is the unfortunate field that affects both." This demo shows precisely what that means. It contains four packages:

 - `dependsdplyr` Depends on dplyr and has no code.
 - `dependsMASS` Depends on MASS and has no code.
 - `dependsdplyr2` Depends on dplyr also, and uses the `select()` function without importing it or the dplyr namespace. Instead, it only `Depends` on dplyr.
 - `importsdplyr` Depends on MASS but imports the `dplyr::select()` function for the sample function as in dependsdplyr2.

This will highlight an important distinction between Depends - which *loads* a package namespace and *attaches* that namespace to the `search()` path - and Imports - which only loads the namespace.


Here's some setup using the installed packages:


```r
devtools::install("dependsdplyr", quick = TRUE, quiet = TRUE)
devtools::install("dependsdplyr2", quick = TRUE, quiet = TRUE)
devtools::install("dependsMASS", quick = TRUE, quiet = TRUE)
devtools::install("importsdplyr", quick = TRUE, quiet = TRUE)
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
## <bytecode: 0x00000000135e53e0>
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


```r
# cleanup
detach("package:dependsMASS")
detach("package:MASS")
detach("package:dependsdplyr2")
detach("package:dplyr")
```

Two further examples might be useful. One is where we Depend on MASS but actually Import from dplyr:


```r
library("importsdplyr")
```

```
## Loading required package: MASS
```

```r
"package:dplyr" %in% search() # FALSE
```

```
## [1] FALSE
```

```r
"package:MASS" %in% search() # TRUE
```

```
## [1] TRUE
```

```r
# imports dplyr function works as expected
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

This highlights that the Depends on MASS is pointless. It's not imported from in NAMESPACE or via `::` so we don't actually need it in importsdplyr and our direct Import from dplyr prevents the attach-order problems of above.

Another example uses Hadley's [conflicted](https://github.com/r-lib/conflicted) package to issue errors on namespace conflicts:


```r
# cleanup last example
detach("package:importsdplyr")
unloadNamespace("importsdplyr")
unloadNamespace("dplyr")
unloadNamespace("MASS")

# load conflicted
library("conflicted")

# try to use importsdplyr again
library("importsdplyr")
```

```
## Loading required package: MASS
```

```r
head(choose_cols(mtcars, "cyl")) # works
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
# and again after loading dplyr and MASS
library("dplyr")
library("MASS")
head(choose_cols(mtcars, "cyl")) # works
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
# but we get the error from conflicted if we use `select()` at the top-level
select(mtcars, cyl)
```

```
## Error: select found in 2 packages. You must indicate which one you want with ::
##  * dplyr::select
##  * MASS::select
```

The lessons learned:

 - Always use a NAMESPACE to specify imports so that your package code isn't harmed by other peoples' use of Depends. (If you don't do this, you'll get a `NOTE` on `R CMD check` but you may not be doing that.)
 - Use Imports to specify any package that must be installed and *loaded* for your package to work. Using Depends may not affect your package code if you're following good practice, but it can affect the user's.
 - Always use a fully qualified reference - `pkg::func()` - when there might be some namespace ambiguity, such as at the top-level or when your package imports from two namespaces that conflict (like MASS and dplyr)
 - Don't use Depends in your packages.

There are exceptions to the final rule (such as needing the methods package), but you never know what your use of Depends might do to someone else's top-level or package code. As always [*Writing R Extensions*](https://cran.r-project.org/doc/manuals/r-devel/R-exts.html#Package-Dependencies) is the best reference for appropriate specification of the DESCRIPTION file.

The main counter-argument I've heard to this is that a developer may have a package that is fairly useless without its strong dependency (e.g., a graphics package building on ggplot2). In these cases, I also think Depends is a bad idea (for all the above reasons) but also because it makes assumptions about the end-user (either that they don't know the dependency or that it's preferable for them to have the package attached.) Smart people disagree here, but my view is that we should try to make as few assumptions as possible about the end-user. Putting the dependency in Imports ensures that the package is installed and its namespace is available. 

I don't think we should further assume that the user wants to be able to use `func()` without having to `library("dependency")` or `dependency::func()`. WRE says graphics extensions are a possible extension but I think it's safer to let users decide whether and when to attach rather than load dependencies. For example, in my own scripts, I generally use `requireNamespace()` and fully qualified references and wouldn't want packages I'm using to attach anything. Lest weird stuff happens:



```r
# really cleanup last example
detach("package:importsdplyr")
unloadNamespace("importsdplyr")
detach("package:dplyr")
unloadNamespace("dplyr")
detach("package:MASS")
unloadNamespace("MASS")
unloadNamespace("conflicted")

# try something from MASS
area(sin, 0, pi) # fails
```

```
## Error in area(sin, 0, pi): could not find function "area"
```

```r
library("importsdplyr")
```

```
## Loading required package: MASS
```

```
## Loading required package: stats
```

```r
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
# try something from MASS, again
area(sin, 0, pi) # works
```

```
## [1] 2
```

Even though we didn't explicitly attach MASS, code from it now works. What changed? We could debug by actively checking `search()` but it's not obvious from the code alone. Again, opinions will differ on whether that's desirable or undesirable in any particular application but to me it feels wrong, since it then affects top-level code in non-obvious ways.

---

Some cleanup:


```r
# remove packages
unloadNamespace("dependsdplyr")
unloadNamespace("dependsdplyr2")
unloadNamespace("dependsMASS")
unloadNamespace("importsdplyr")
remove.packages(c("dependsdplyr", "dependsdplyr2", "dependsMASS", "importsdplyr"))
```

```
## Removing packages from 'C:/Program Files/R/R-3.5.0/library'
## (as 'lib' is unspecified)
```

```r
# session info
sessionInfo()
```

```
## R version 3.5.0 (2018-04-23)
## Platform: x86_64-w64-mingw32/x64 (64-bit)
## Running under: Windows 7 x64 (build 7601) Service Pack 1
## 
## Matrix products: default
## 
## locale:
## [1] LC_COLLATE=English_United Kingdom.1252 
## [2] LC_CTYPE=English_United Kingdom.1252   
## [3] LC_MONETARY=English_United Kingdom.1252
## [4] LC_NUMERIC=C                           
## [5] LC_TIME=English_United Kingdom.1252    
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
## [1] MASS_7.3-49      conflicted_0.1.0
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.16     knitr_1.20       bindr_0.1.1      magrittr_1.5    
##  [5] devtools_1.13.5  tidyselect_0.2.4 R6_2.2.2         rlang_0.2.0     
##  [9] dplyr_0.7.5      stringr_1.3.1    tools_3.5.0      git2r_0.21.0    
## [13] withr_2.1.2      assertthat_0.2.0 digest_0.6.15    tibble_1.4.2    
## [17] crayon_1.3.4     bindrcpp_0.2.2   purrr_0.2.4      memoise_1.1.0   
## [21] glue_1.2.0       evaluate_0.10.1  stringi_1.1.7    compiler_3.5.0  
## [25] pillar_1.2.2     pkgconfig_2.0.1
```
