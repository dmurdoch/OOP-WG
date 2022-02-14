
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Object-oriented Programming Working Group

-   [Initial proposal](proposal/proposal.org)
-   [Requirements brainstorming](spec/requirements.md)
-   [Minutes](minutes/)
-   [Code](R/) (this repository is an R package)

These ideas have been implemented in the R7 package, hosted in this
repository.

<!-- badges: start -->

[![R-CMD-check](https://github.com/RConsortium/OOP-WG/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/RConsortium/OOP-WG/actions/workflows/R-CMD-check.yaml)
[![Codecov test
coverage](https://codecov.io/gh/RConsortium/OOP-WG/branch/master/graph/badge.svg)](https://codecov.io/gh/RConsortium/OOP-WG?branch=master)
<!-- badges: end -->

## Classes and objects

``` r
library(R7)

range <- new_class("range",
  constructor = function(start, end) {
    new_object(start = start, end = end)
  },
  validator = function(x) {
    if (x@end < x@start) {
      "<range>@end must be greater than or equal to <range>@start"
    }
  },
  properties = list(
    start = "numeric",
    end = "numeric",
    new_property(
      name = "length",
      class = "numeric",
      getter = function(x) x@end - x@start,
      setter = function(x, value) {
        x@end <- x@start + value
        x
      }
    )
  )
)

x <- range(start = 1, end = 10)

x@start
#> [1] 1

x@end
#> [1] 10

x@length
#> [1] 9

x@length <- 5

x@length
#> [1] 5

# incorrect properties throws an error
x@middle
#> Error in prop(object, name): Can't find property <range>@middle

# assigning properties verifies the class matches the class of the value
x@end <- "foo"
#> Error: <range>@end must be of class <integer> or <double>, not <character>

# assigning properties runs the validator
x@end <- 0
#> Error: <range> object is invalid:
#> - <range>@end must be greater than or equal to <range>@start

# Print methods for both R7_class objects
object_class(x)
#> <R7_class>
#> @ name  :  range
#> @ parent: <R7_object>
#> @ properties:
#>  $ start : <integer> or <double>
#>  $ end   : <integer> or <double>
#>  $ length: <integer> or <double>

# As well as normal R7_objects
x
#> <range/R7_object>
#> @ start :  num 1
#> @ end   :  num 6
#> @ length:  num 5
```

## Generics and methods

``` r
text <- new_class("text", parent = "character")
foo <- new_generic("foo", "x")
method(foo, text) <- function(x, ...) paste0("foo-", x)

foo(text("hi"))
#> [1] "foo-hi"
```

## Multiple dispatch

Multiple dispatch uses a table stored in the `methods` property of the
generic. This table is a nested set of hashed environments based on the
classes of the methods. e.g.

For `method(foo, c("character", "numeric"))` the method would be stored
at `foo@methods[["character"]][["numeric"]]`.

At each level the search iteratively searches along objects class
vector.

``` r
bar <- new_generic("bar", c("x", "y"))
method(bar, list("character", "double")) <- function(x, y) paste0("foo-", x, ":", y)

bar("hi", 42)
#> [1] "foo-hi:42"
```

## Calling the next method

`next_method()` is used to call the next method for the arguments. This
works by looking up the call stack and retrieving R7 methods which have
already been called, then doing a method search with those methods
excluded. This ensures you cannot call the same method twice.

``` r
method(bar, list(text, "double")) <- function(x, y, ...) {
  res <- next_method()(x, y)
  paste0("2 ", res)
}

bar(text("hi"), 42)
#> [1] "2 foo-hi:42"
```

## Non-standard evaluation

`method_call()` retains promises for dispatch arguments in basically the
same way as `UseMethod()`, so non-standard evaluation works basically
the same as S3.

``` r
subset2 <- new_generic("subset2", "x")

method(subset2, s3_class("data.frame")) <- function(x, subset = NULL, select = NULL, drop = FALSE) {
  e <- substitute(subset)
  # Unlike S3, R7 creates a frame for the generic, so we need to
  # go one extra level up to get to the user's evaluation environment
  r <- eval(e, x, parent.frame(2))
  r <- r & !is.na(r)
  nl <- as.list(seq_along(x))
  names(nl) <- names(x)
  vars <- eval(substitute(select), nl, parent.frame())
  x[r, vars, drop = drop]
}

subset2(mtcars, hp > 200, c(wt, qsec))
#>                        wt  qsec
#> Duster 360          3.570 15.84
#> Cadillac Fleetwood  5.250 17.98
#> Lincoln Continental 5.424 17.82
#> Chrysler Imperial   5.345 17.42
#> Camaro Z28          3.840 15.41
#> Ford Pantera L      3.170 14.50
#> Maserati Bora       3.570 14.60
```

### External generics

If you want to define methods for R7 generics defined in another package
you can use `new_extrenal_generic` to declare the external generic, then
add `R7::external_methods_register()` to the `.onLoad` function in your
package. `external_methods_register()` will automatically setup on-load
hooks for ‘soft’ dependencies in `Suggests` so the method will be added
when the dependency is eventually loaded.

``` r
.onLoad <- function(libname, pkgname) {
  R7::external_methods_register()
}

foo <- new_external_generic("pkg1", "foo")
method(foo, "integer") <- function(x) paste0("foo-", x)
```

## Design workflow

-   File an issue to discuss the topic and build consensus.
-   Once consensus has been reached, the issue author should create a
    pull request that summarises the discussion in the appropriate `.md`
    file, and request review from all folks who participated the issue
    discussion.
-   Once all participants have accepted the PR, the original author
    merges.
