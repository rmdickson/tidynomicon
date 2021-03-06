---
title: "Object-Oriented Programming"
output: md_document
permalink: /oop/
questions:
  - "How can I do object-oriented programming in R?"
  - "How do I specify an object's class?"
  - "How do I provide methods for a class?"
  - "How should I create objects of a class I have defined?"
objectives:
  - "Correctly identify the most commonly used object-oriented programming system in R."
  - "Explain what attributes R and correctly set and query objects' attributes, class, and dimensions."
  - "Explain how to define a new method for a class."
  - "Describe and implement the three functions that should be written for any user-defined class."
keypoints:
  - "S3 is the most commonly used object-oriented programming system in R."
  - "Every object can store metadata about itself in attributes, which are set and queried with `attr`."
  - "The `dim` attribute stores the dimensions of a matrix (which is physically stored as a vector)."
  - "The `class` attribute of an object defines its class or classes (it may have several character entries)."
  - "When `F(X, ...)` is called, and `X` has class `C`, R looks for a function called `F.C` (the `.` is just a naming convention)."
  - "If an object has multiple classes in its `class` attribute, R looks for a corresponding method for each in turn."
  - "Every user defined class `C` should have functions `new_C` (to create it), `validate_C` (to validate its integrity), and `C` (to create and validate)."
---



Programmers spend a great deal of their time trying to create order out of chaos,
and the rest of their time inventing new ways to create more chaos.
Object-oriented programming serves both needs well:
it allows good software designers to create marvels,
and less conscientious or experienced ones to create horrors.

R has not one, not two, but at least three different frameworks for object-oriented programming.
By far the most widely used is known as [S3](#S3)
(because it was first introduced with Version 3 of S,
the language from which R is derived).
Unlike the approaches used in Java, Python, and similarly pedestrian languages,
S3 does not require users to define classes.
Instead,
they add [attributes](../glossary/#attribute) to data,
then write specialized version of [generic functions](../glossary/#generic-function)
to process data identified by those attributes.
Since attributes can be used in other ways as well,
we will start by exploring them.

## Attributes

Let's begin by creating a matrix containing the first few hundreds:


```r
values <- 100 * 1:9 # creates c(100, 200, ..., 900)
m <- matrix(values, nrow = 3, ncol = 3)
m
#>      [,1] [,2] [,3]
#> [1,]  100  400  700
#> [2,]  200  500  800
#> [3,]  300  600  900
```

Behind the scenes,
R continues to store our nine values as a vector.
However,
it adds an attribute called `class` to the vector to identify it as a matrix:


```r
class(m)
#> [1] "matrix"
```

and another attribute called `dim` to store its dimensions as a 2-element vector:


```r
dim(m)
#> [1] 3 3
```

An object's attributes are simply a set of name-value pairs;
we can find out what attributes are present using `attributes`,
and show or set individual attributes using `attr`:


```r
attr(m, "prospects") <- "dismal"
attributes(m)
#> $dim
#> [1] 3 3
#> 
#> $prospects
#> [1] "dismal"
```

What are the type and attributes of a tibble?


```r
t <- tribble(
  ~a, ~b,
  1, 2,
  3, 4)
typeof(t)
#> [1] "list"
attributes(t)
#> $names
#> [1] "a" "b"
#> 
#> $row.names
#> [1] 1 2
#> 
#> $class
#> [1] "tbl_df"     "tbl"        "data.frame"
```

This tells us that a tibble is stored as a list (the first line of output),
that it has an attribute called `names` that stores the names of its columns,
another called `row.names` that stores the names of its rows (a feature we should ignore),
and finally three classes.
These classes tell R what functions to search for when we are (for example)
asking for the length of a tibble (which is the number of rows it contains):


```r
length(t)
#> [1] 2
```

## Classes

To show how classes and generic functions work together,
let's customize the way that 2D coordinates are converted to strings.
First,
we'll create two coordinate vectors:


```r
first <- c(0.5, 0.7)
class(first) <- "two_d"
print(first)
#> [1] 0.5 0.7
#> attr(,"class")
#> [1] "two_d"
second <- c(1.3, 3.1)
class(second) <- "two_d"
print(second)
#> [1] 1.3 3.1
#> attr(,"class")
#> [1] "two_d"
```

Separately, let's define the behavior of `toString` for such objects:


```r
toString.two_d <- function(obj){
  paste0("<", obj[1], ", ", obj[2], ">")
}
toString(first)
#> [1] "<0.5, 0.7>"
toString(second)
#> [1] "<1.3, 3.1>"
```

S3's protocol is simple:
given a function F and an object whose class is C,
it looks for a function named F.C.
If it doesn't find one,
it looks at the object's next class (assuming it has more than one);
once its user-assigned classes are exhausted,
it uses whatever function the system has defined for its base type (in this case, character vector).
We can trace this process by importing the sloop package and calling `s3_dispatch`:


```r
library(sloop)
#> 
#> Attaching package: 'sloop'
#> The following objects are masked from 'package:pryr':
#> 
#>     ftype, is_s3_generic, is_s3_method, otype
s3_dispatch(toString(first))
#> => toString.two_d
#>  * toString.default
```

Compare this with calling `toString` on a plain old character vector:


```r
s3_dispatch(toString(c(7.1, 7.2)))
#>    toString.double
#>    toString.numeric
#> => toString.default
```

The specialized functions associated with a generic function like `toString` are called [methods](../glossary/#method).
Unlike languages that require methods to be defined all together as part of a class,
S3 allows us to add methods when and as we see fit.
But that doesn't mean we should:
minds confined to three dimensions of space and one of time are simply not capable of comprehending
the staggering complexity that can result from doing so.
Instead,
we should always write three functions that work together for a class like `prospects`:

- A [constructor](../glossary/#constructor) called `new_two_d`
  that creates objects of our class.
- An optional [validator](../glossary/#validator) called `validate_two_d`
  that checks the consistency and correctness of an object's values.
- An optional [helper](../glossary/#helper), simply called `two_d`,
  that most users will call to create and validate objects.

The constructor's first argument should always be the base object (in our case, the two-element vector).
It should also have one argument for each attribute the object is to have, if any.
Unlike matrices, our 2D points don't have any extra arguments, so our constructor needs no extra arguments.
Crucially,
the constructor checks the type of its arguments to ensure that the object has at least some chance of being valid.


```r
new_two_d <- function(coordinates){
  stopifnot(is.numeric(coordinates))
  class(coordinates) <- "two_d"
  coordinates
}

example <- new_two_d(c(4.4, -2.2))
toString(example)
#> [1] "<4.4, -2.2>"
```

Validators are only needed when checks on data correctness and consistency are expensive.
For example,
if we were to define a class to represent sorted vectors,
checking that each element is no less than its predecessor could take a long time for very long vectors.
To illustrate this,
we will check that we have exactly two coordinates;
in real code,
we would probably include this (inexpensive) check in the constructor.


```r
validate_two_d <- function(coordinates) {
  stopifnot(length(coordinates) == 2)
  stopifnot(class(coordinates) == "two_d")
}

validate_two_d(example)    # should succeed silently
validate_two_d(c(1, 3))    # should fail
#> Error in validate_two_d(c(1, 3)): class(coordinates) == "two_d" is not TRUE
validate_two_d(c(2, 2, 2)) # should also fail
#> Error in validate_two_d(c(2, 2, 2)): length(coordinates) == 2 is not TRUE
```

The third and final function in our trio is the helper that provides a user-friendly interface to construction of our class.
It should call the constructor and the validator (if one exists),
but should also provide a richer set of defaults,
better error messages,
and so on.
Purely for illustrative purposes,
we shall allow the user to provide either one argument (which must be a two-element vector)
or two (which must each be numeric):


```r
two_d <- function(...){
  args <- list(...)
  if (length(args) == 1) {
    args <- args[[1]]    # extract original value
  }
  else if (length(args) == 2) {
    args <- unlist(args) # convert list to vector
  }
  result <- new_two_d(args)
  validate_two_d(result)
  result
}

here <- two_d(10.1, 11.2)
toString(here)
#> [1] "<10.1, 11.2>"
there <- two_d(c(15.6, 16.7))
toString(there)
#> [1] "<15.6, 16.7>"
```

## Inheritance

We said above that an object can have more than one class,
and that S3 searches the classes in order when it wants to find a method to call.
Methods can also trigger invocation of other methods explicitly in order to supplement,
rather than replace,
the behavior of other classes.
To explore this,
we shall look at that classic of object-oriented design, shapes---the safe kind, of course,
not those whose non-Euclidean angles have placed such intolerable stress on the minds of so many of our colleagues over the years.


```r
new_polygon <- function(coords, name) {
  points <- map(coords, two_d)
  class(points) <- "polygon"
  attr(points, "name") <- name
  points
}

toString.polygon <- function(poly) {
  paste0(attr(poly, "name"), ": ", paste0(map(poly, toString), collapse = ", "))
}

right <- new_polygon(list(c(0, 0), c(1, 0), c(0, 1)), "triangle")
toString(right)
#> [1] "triangle: 0, 0, 1, 0, 0, 1"
```

Now we will add colored shapes:


```r
new_colored_polygon <- function(coords, name, color) {
  object <- new_polygon(coords, name)
  attr(object, "color") <- color
  class(object) <- c("colored_polygon", class(object))
  object
}

pinkish <- new_colored_polygon(list(c(0, 0), c(1, 0), c(1, 1)), "triangle", "roseate")
class(pinkish)
#> [1] "colored_polygon" "polygon"
toString(pinkish)
#> [1] "triangle: 0, 0, 1, 0, 1, 1"
```

So far so good:
since we have not defined a method to handle colored polygons specifically,
we get the behavior for a regular polygon.
Let's add another method:


```r
toString.colored_polygon <- function(poly) {
  paste0(toString.polygon(poly), "+ color = ", attr(poly, "color"))
}

toString(pinkish)
#> [1] "triangle: 0, 0, 1, 0, 1, 1+ color = roseate"
```

In practice,
we will almost always place all of the methods associated with a class in the same file as its constructor, validator, and helper.
The time has finally come for us to explore projects and packages.

{% include links.md %}
