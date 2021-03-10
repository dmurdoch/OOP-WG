
<!-- README.md is generated from README.Rmd. Please edit that file -->

# Object-oriented Programming Working Group

  - [Initial proposal](proposal/proposal.org)
  - [Requirements brainstorming](spec/requirements.md)
  - [Minutes](minutes/)
  - [Code](R/) (this repository is an R package)

These ideas have been implemented in the R7 package, hosted in this
repository.

<!-- badges: start -->

[![R-CMD-check](https://github.com/RConsortium/OOP-WG/workflows/R-CMD-check/badge.svg)](https://github.com/RConsortium/OOP-WG/actions)
[![Codecov test
coverage](https://codecov.io/gh/RConsortium/OOP-WG/branch/master/graph/badge.svg)](https://codecov.io/gh/RConsortium/OOP-WG?branch=master)
<!-- badges: end -->

## Classes and objects

``` r
library(R7)

range <- class_new("range",
  constructor = function(start, end) {
    object_new(start = start, end = end)
  },
  validator = function(x) {
    if (x@end < x@start) {
      "`end` must be greater than or equal to `start`"
    }
  },
  properties = list(
    start = "numeric",
    end = "numeric",
    property_new(
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
#> Error: Can't find property 'middle' in <range>

# assigning properties verifies the class matches the class of the value
x@end <- "foo"
#> Error: `value` must be of class <numeric>:
#> - `value` is of class <character>

# assigning properties runs the validator
x@end <- 0
#> Error: Invalid <range> object:
#> - `end` must be greater than or equal to `start`

# Print methods for both R7_class objects
object_class(x)
#> <R7_class>
#> @name range
#> @parent <R7_object>
#> @properties
#>  $start  <numeric>
#>  $end    <numeric>
#>  $length <numeric>

# As well as normal R7_objects
x
#> <R7_object> <range>
#> @start  1
#> @end    6
#> @length 5

# Use `.data` to refer to and retrieve the base data type, properties are
# automatically removed, but non-property attributes (such as names) are retained.

text <- class_new("text", parent = "character", constructor = function(text) object_new(.data = text))

y <- text(c(foo = "bar"))

str(y@.data)
#>  Named chr "bar"
#>  - attr(*, "names")= chr "foo"
```

## Generics and methods

``` r
text <- class_new("text", parent = "character", constructor = function(text) object_new(.data = text))

foo <- generic_new(name = "foo", signature = "x")

method_new(foo, list("text"), function(x) paste0("foo-", x))

foo(text("hi"))
#> [1] "foo-hi"
```

## Multiple dispatch

``` r
number <- class_new("number", parent = "numeric", constructor = function(x) object_new(.data = x))

bar <- generic_new(name = "bar", signature = c("x", "y"))

method_new(bar, list("character", "numeric"), function(x, y) paste0("foo-", x, ":", y))

bar(text("hi"), number(42))
#> [1] "foo-hi:42"
```

## Calling the next method

``` r
method_new(bar, list("text", "number"), function(x, y) {
  res <- method_next()(x, y)
  paste0("2 ", res)
})

bar(text("hi"), number(42))
#> [1] "2 foo-hi:42"
```

### Load time registration

``` r
.onLoad <- function(libname, pkgname) {
  R7::method_register()
}

method_new("pkg1::foo", list("text", "numeric"), function(x, y) paste0("foo-", x, ": ", y))
```

## Performance

The dispatch performance should be roughly on par with S3 and S4, though
as this is implemented in the package there is some overhead due to
`.Call` vs `.Primitive`.

Dispatch uses a table stored in the `methods` property of the generic.
This table is a nested set of hashed environments based on the classes
of the methods. e.g.

`method(foo, c("character", "numeric"))` method would be stored at

`foo@methods[["character"]][["numeric"]]`

At each level the search iteratively searches up the class vector for
the object.

``` r
text <- class_new("text", parent = "character", constructor = function(text) object_new(.data = text))
number <- class_new("number", parent = "numeric", constructor = function(x) object_new(.data = x))

x <- text("hi")
y <- number(1)

foo_R7 <- generic_new(name = "foo_R7", signature = "x")
method(foo_R7, "text") <- function(x) paste0(x, "-foo")

foo_s3 <- function(x) {
  UseMethod("foo_s3")
}

foo_s3.text <- function(x) {
  paste0(x, "-foo")
}

library(methods)
setOldClass(c("number", "numeric", "R7_object"))
setOldClass(c("text", "character", "R7_object"))

setGeneric("foo_s4", function(x) standardGeneric("foo_s4"))
#> [1] "foo_s4"
setMethod("foo_s4", c("text"), function(x) paste0(x, "-foo"))

# Measure performance of single dispatch
bench::mark(foo_R7(x), foo_s3(x), foo_s4(x))
#> # A tibble: 3 x 6
#>   expression      min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 foo_R7(x)    5.97µs   8.76µs   115512.        0B     11.6
#> 2 foo_s3(x)    3.91µs   4.32µs   210119.        0B     21.0
#> 3 foo_s4(x)    4.03µs   4.49µs   201600.        0B     20.2

bar_R7 <- generic_new("bar_R7", c("x", "y"))
method(bar_R7, list("text", "number")) <- function(x, y) paste0(x, "-", y, "-bar")

setGeneric("bar_s4", function(x, y) standardGeneric("bar_s4"))
#> [1] "bar_s4"
setMethod("bar_s4", c("text", "number"), function(x, y) paste0(x, "-", y, "-bar"))

# Measure performance of double dispatch
bench::mark(bar_R7(x, y), bar_s4(x, y))
#> # A tibble: 2 x 6
#>   expression        min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr>   <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 bar_R7(x, y)  12.36µs   15.3µs    65846.        0B     19.8
#> 2 bar_s4(x, y)   9.61µs   10.9µs    85140.        0B     17.0
```

A potential optimization is caching based on the class names, but lookup
should be fast without this.

The following benchmark generates a class heiarchy of different levels
and lengths of class names and compares the time to dispatch on the
first class in the hiearchy vs the time to dispatch on the last class.

We find that even in very extreme cases (e.g. 100 deep heirachy 100 of
character class names) the overhead is reasonable, and for more
reasonable cases (e.g. 10 deep hiearchy of 15 character class names) the
overhead is basically negligible.

``` r
library(R7)

gen_character <- function (n, min = 5, max = 25, values = c(letters, LETTERS, 0:9)) {
  lengths <- sample(min:max, replace = TRUE, size = n)
  values <- sample(values, sum(lengths), replace = TRUE)
  starts <- c(1, cumsum(lengths)[-n] + 1)
  ends <- cumsum(lengths)
  mapply(function(start, end) paste0(values[start:end], collapse=""), starts, ends)
}

bench::press(
  num_classes = c(3, 5, 10, 50, 100),
  class_size = c(15, 100),
  {
    # Construct a class hierarchy with that number of classes
    text <- class_new("text", parent = "character", constructor = function(text) object_new(.data = text))
    parent <- text
    classes <- gen_character(num_classes, min = class_size, max = class_size)
    for (x in classes) {
      assign(x, class_new(x, parent = parent, constructor = function(text) object_new(.data = text)))
      parent <- get(x)
    }

    # Get the last defined class
    cls <- parent

    # Construct an object of that class
    x <- do.call(cls, list("hi"))

    # Define a generic and a method for the last class (best case scenario)
    foo_R7 <- generic_new(name = "foo_R7", signature = "x")
    method(foo_R7, cls) <- function(x) paste0(x, "-foo")

    # Define a generic and a method for the first class (worst case scenario)
    foo2_R7 <- generic_new(name = "foo2_R7", signature = "x")
    method(foo2_R7, R7_object) <- function(x) paste0(x, "-foo")

    bench::mark(
      best = foo_R7(x),
      worst = foo2_R7(x)
    )
  }
)
#> # A tibble: 20 x 8
#>    expression num_classes class_size      min   median `itr/sec` mem_alloc `gc/sec`
#>    <bch:expr>       <dbl>      <dbl> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#>  1 best                 3         15    6.4µs   7.62µs   128108.        0B    25.6 
#>  2 worst                3         15   6.71µs    8.4µs   119461.        0B    11.9 
#>  3 best                 5         15   6.54µs    8.1µs   120687.        0B    24.1 
#>  4 worst                5         15   7.33µs   8.64µs   108336.        0B    10.8 
#>  5 best                10         15   6.46µs   8.05µs   120115.        0B    24.0 
#>  6 worst               10         15   7.44µs    8.1µs   105869.        0B    21.2 
#>  7 best                50         15   7.38µs   8.17µs   111123.        0B    11.1 
#>  8 worst               50         15  11.17µs  12.05µs    77646.        0B    15.5 
#>  9 best               100         15   8.03µs   8.74µs   104572.        0B    20.9 
#> 10 worst              100         15  15.88µs  16.82µs    55421.        0B    11.1 
#> 11 best                 3        100   6.49µs   7.92µs   122183.        0B    12.2 
#> 12 worst                3        100   6.99µs   8.61µs   115842.        0B    23.2 
#> 13 best                 5        100   6.79µs   8.05µs   113270.        0B    22.7 
#> 14 worst                5        100   7.25µs   9.44µs   104523.        0B    10.5 
#> 15 best                10        100   6.75µs    7.5µs   119261.        0B    23.9 
#> 16 worst               10        100   8.07µs   9.55µs    99423.        0B    19.9 
#> 17 best                50        100   7.22µs   8.66µs   112672.        0B    11.3 
#> 18 worst               50        100  14.28µs  15.37µs    60045.        0B    12.0 
#> 19 best               100        100   8.01µs   8.88µs   103941.        0B    20.8 
#> 20 worst              100        100  22.52µs  24.05µs    37205.        0B     3.72
```

And the same benchmark using double-dispatch vs single dispatch

``` r
bench::press(
  num_classes = c(3, 5, 10, 50, 100),
  class_size = c(15, 100),
  {
    # Construct a class hierarchy with that number of classes
    text <- class_new("text", parent = "character", constructor = function(text) object_new(.data = text))
    parent <- text
    classes <- gen_character(num_classes, min = class_size, max = class_size)
    for (x in classes) {
      assign(x, class_new(x, parent = parent, constructor = function(text) object_new(.data = text)))
      parent <- get(x)
    }

    # Get the last defined class
    cls <- parent

    # Construct an object of that class
    x <- do.call(cls, list("hi"))
    y <- do.call(cls, list("ho"))

    # Define a generic and a method for the last class (best case scenario)
    foo_R7 <- generic_new(name = "foo_R7", signature = c("x", "y"))
    method(foo_R7, list(cls, cls)) <- function(x, y) paste0(x, y, "-foo")

    # Define a generic and a method for the first class (worst case scenario)
    foo2_R7 <- generic_new(name = "foo2_R7", signature = c("x", "y"))
    method(foo2_R7, list(R7_object, R7_object)) <- function(x, y) paste0(x, y, "-foo")

    bench::mark(
      best = foo_R7(x, y),
      worst = foo2_R7(x, y)
    )
  }
)
#> # A tibble: 20 x 8
#>    expression num_classes class_size      min   median `itr/sec` mem_alloc `gc/sec`
#>    <bch:expr>       <dbl>      <dbl> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#>  1 best                 3         15   7.94µs   9.94µs    99304.        0B    19.9 
#>  2 worst                3         15   8.35µs  10.57µs    96043.        0B    28.8 
#>  3 best                 5         15   7.92µs   9.97µs    99115.        0B    19.8 
#>  4 worst                5         15   8.93µs  10.98µs    90284.        0B    18.1 
#>  5 best                10         15   7.92µs   9.94µs    98710.        0B    29.6 
#>  6 worst               10         15   9.58µs  11.93µs    83590.        0B    16.7 
#>  7 best                50         15   9.34µs  10.98µs    86442.        0B    17.3 
#>  8 worst               50         15  17.53µs  18.63µs    48850.        0B    14.7 
#>  9 best               100         15   10.9µs  12.64µs    73649.        0B    14.7 
#> 10 worst              100         15  28.04µs  29.67µs    31080.        0B     9.33
#> 11 best                 3        100   7.85µs   9.87µs   100076.        0B    20.0 
#> 12 worst                3        100   8.89µs  10.92µs    90573.        0B    18.1 
#> 13 best                 5        100   8.18µs  10.39µs    93604.        0B    28.1 
#> 14 worst                5        100   9.59µs     12µs    82921.        0B    16.6 
#> 15 best                10        100   8.34µs  10.29µs    95143.        0B    19.0 
#> 16 worst               10        100  11.14µs  12.74µs    74153.        0B    22.3 
#> 17 best                50        100    9.2µs  11.34µs    87691.        0B    17.5 
#> 18 worst               50        100   24.4µs  25.67µs    36345.        0B    10.9 
#> 19 best               100        100  10.77µs  13.41µs    74689.        0B    14.9 
#> 20 worst              100        100   40.7µs  42.28µs    21972.        0B     4.40
```

## Questions

  - Best way to support `substitute()` calls in methods? We need to
    evaluate the argument promises to do the dispatch, but we want to
    pass the un-evaluated promise to the call?
  - Should we remove `method()<-`, We can’t do the following, really
    want to keep this syntax if possible.
      - can’t use `method("foo")<-`
      - can’t use `method(otherpkg::foo)<-`
      - can’t use `method("otherpkg::foo")<-`
  - What should happen if you call `method_new()` on a S3 generic?
    1.  Should we create a new R7 generic out of the S3 generic?
    2.  Or just register the R7 object using `registerS3method()`? ++

## Design workflow

  - File an issue to discuss the topic and build consensus.
  - Once consensus has been reached, the issue author should create a
    pull request that summarises the discussion in the appropriate `.md`
    file, and request review from all folks who participated the issue
    discussion.
  - Once all participants have accepted the PR, the original author
    merges.

## TODO

  - Objects
      - [x] - A class object attribute, a reference to the class object,
        and retrieved with `object_class()`.
      - [x] - For S3 compatibility, a class attribute, a character
        vector of class names.
      - [x] - Additional attributes storing properties defined by the
        class, accessible with `@/property()`.
  - Classes
      - [x] - R7 classes are first class objects with the following
          - [x] - `name`, a human-meaningful descriptor for the class.
          - [x] - `parent`, the class object of the parent class.
          - [x] - A constructor, an user-facing function used to create
            new objects of this class. It always ends with a call to
            `object_new()` to initialize the class.
          - [x] - A validator, a function that takes the object and
            returns NULL if the object is valid, otherwise a character
            vector of error messages.
          - [x] - properties, a list of property objects
  - Initialization
      - [x] - The constructor uses `object_new()` to initialize a new
        object, this
          - [x] - Inspects the enclosing scope to find the “current”
            class.
          - [ ] - Creates the prototype, by either by calling the parent
            constructor or by creating a base type and adding class and
            `object_class()` attributes to it.
          - [x] - Validates properties then adds to prototype.
          - [x] - Validates the complete object.
  - Shortcuts
      - [ ] - any argument that takes a class object can instead take
        the name of a class object as a string
      - [x] - instead of providing a list of property objects, you can
        instead provide a named character vector.
  - Validation
      - [x] - valid\_eventually
      - [x] - valid\_implicitly
  - Unions
      - [ ] - Used in properties to allow a property to be one of a set
        of classes
      - [x] - In method dispatch as a convenience for defining a method
        for multiple classes
  - Properties
      - [x] - Accessed using `property()` / `property<-`
      - [x] - Accessed using `@` / `@<-`
      - [x] - A name, used to label output
      - [ ] - A optional class or union
      - [x] - An optional accessor functions, both getter and setters
      - [ ] - Properties are created with `prop_new()`
  - Generics
      - [x] - It knows its name and the names of the arguments in its
        signature
      - [x] - Calling `generic_new()` defines a new generic
      - [ ] - By convention, any argument that takes a generic function,
        can instead take the name of a generic function supplied as a
        string
  - Methods
      - Registration
          - [x] - Methods are defined by calling method\<-(generic,
            signature, method):
          - [x] - generic is a generic function.
          - [x] - signature is a
              - [x] - single class object
              - [x] - a class union
              - [x] - list of class objects/unions
              - [x] - a character vector.
          - [ ] - method is a compatible function
          - [x] - `method_new` is designed to work at run-time
              - [ ] - `method_new` should optionally take a package
                version, so the method is only registered if the package
                is newer than the version.
          - [ ] - Can define methods where one of the arguments is
            missing
          - [ ] - Can define methods where one of the arguments has any
            type
      - Dispatch
          - [x] - Dispatch is nested, meaning that if there are multiple
            arguments in the generic signature, it will dispatch on the
            first argument, then the second.
          - [x] - A `plot()` generic dispatching on `x`, e.g. `plot <-
            function(x) { method(plot, object_class(x))(x) }`
          - [x] - A `publish()` that publishes an object `x` to a
            destination `y`, dispatching on both arguments,
            e.g. `publish <- function(x, y, ...) { method(publish,
            list(object_class(x), object_class(y)))(x, y, ...) }`
          - [x] - `...` is not used for dispatch
          - [x] - R7 generics can dispatch with base type objects
          - [x] - R7 generics can dispatch with S3 objects
          - [x] - R7 generics can dispatch with S4 objects
          - [x] - `method_next()` can dispatch on multiple arguments,
            avoiding methods that have already been called.
  - Compatibility
      - S3
          - [x] - Since the class attribute has the same semantics as
            S3, S3 dispatch should be fully compatible.
          - [x] - The new generics should also be able to handle legacy
            S3 objects.
          - [x] - `method()` falls back to single argument S3 dispatch
            if the R7 dispatch fails.
          - [ ] - `method()` uses S3 group generics as well
      - S4
          - [x] - Since the new generics will fallback to S3 dispatch,
            they should support S4 objects just as S3 generics support
            them now.
  - Documentation
      - [ ] - Generate index pages that list the methods for a generic
        or the methods with a particular class in their signature
