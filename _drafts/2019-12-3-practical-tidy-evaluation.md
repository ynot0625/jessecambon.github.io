---
layout: post
title:  "Practical Tidy Evaluation"
date:   2019-12-3
tags: test
output: 
  md_document:
    variant: gfm
    preserve_yaml: TRUE
---

[Tidy evaluation](https://tidyeval.tidyverse.org/) is a framework for
controlling how expressions and variables in your code are evaluated by
[tidyverse](https://www.tidyverse.org/) functions. This framework,
housed in the [rlang package](https://rlang.r-lib.org), is a powerful
tool for writing more efficient and modular tidyverse code. In
particular, you’ll find it useful for passing variable names as inputs
to functions that use tidyverse packages like
[dplyr](https://dplyr.tidyverse.org/) and
[ggplot2](https://ggplot2.tidyverse.org/).

The goal of this post is to offer accessible examples and intuition for
putting tidy evaluation to work in your own code. Because of this, I
will keep conceptual explanations brief, but for more comprehensive
documentation you can refer to [rlang’s
website](https://rlang.r-lib.org/reference/), the [‘Tidy Evaluation’
book](https://tidyeval.tidyverse.org/) by Lionel Henry and Hadley
Wickham, and the [Metaprogramming Section of the ‘Advanced R’
book](https://adv-r.hadley.nz/metaprogramming.html) by Hadley Wickham.

### Motivating Example

To begin, let’s consider a basic analysis of the [mtcars
dataset](https://stat.ethz.ch/R-manual/R-devel/library/datasets/html/mtcars.html).
Below we calculate maximum and minimum horsepower (hp) by the number of
cylinders (cyl) using the
[group\_by](https://dplyr.tidyverse.org/reference/group_by.html) and
[summarize](https://dplyr.tidyverse.org/reference/summarise.html)
functions from [dplyr](https://dplyr.tidyverse.org/).

``` r
library(dplyr)
hp_by_cyl <- mtcars %>% 
  group_by(cyl) %>%
  summarize(max_hp=max(hp),
            min_hp=min(hp))
```

| cyl | max\_hp | min\_hp |
| --: | ------: | ------: |
|   4 |     113 |      52 |
|   6 |     175 |     105 |
|   8 |     335 |     150 |

Now let’s say we wanted to repeat this calculation multiple times while
changing which variable we group by. A brute force method to accomplish
this would be to copy and paste our code as many times as necessary and
replace “cyl” with our new variable names. However that’s inefficient
especially if our code gets more complicated, requires many iterations,
or will need to be updated frequently.

To avoid this inelegant solution you might think to store the name of a
variable inside of another variable like this `groupby_var <- "vs"`.
Then you could attempt to use your newly created “groupby\_var” variable
in your code: `group_by(groupby_var)`. However, if you try this you will
find it doesn’t work. The “group\_by” function expects the name of the
variable you want to group by as an input, not the name of a variable
that *contains* the name of the variable you want to group by.

This is the kind of headache that tidy evaluation can help you remedy.
In the example below we use the
[quo](https://rlang.r-lib.org/reference/quotation.html) function and the
“bang-bang” [\!\!](https://rlang.r-lib.org/reference/nse-force.html)
operator to set “vs” (engine type, 0 = automatic, 1 = manual) as our
group by variable. The quo function allows us to store the variable name
and “\!\!” extracts the stored variable name.

``` r
groupby_var <- quo(vs)

hp_by_vs <- mtcars %>% 
  group_by(!!groupby_var) %>%
  summarize(max_hp=max(hp),
            min_hp=min(hp))
```

| vs | max\_hp | min\_hp |
| -: | ------: | ------: |
|  0 |     335 |      91 |
|  1 |     123 |      52 |

The code above provides a method for setting the group by variable by
modifying the input to the “quo” function. This has some utility,
particularly if we were referencing the group by variable multiple
times. However, if we want to utilize a piece of code like this
repeatedly in a script then we should consider packaging it into a
function which is what we will do next.

### Making Functions with Tidy Evaluation

To use tidy evaluation in a function, we will still use the “\!\!”
operator as we did above, but instead of “quo” we will use the
[enquo](https://rlang.r-lib.org/reference/nse-defuse.html) function. Our
function takes the group by variable and the measurement variable as
inputs so that we can now calculate maximum and minimum values of other
variables.

I’ve also introduced the “walrus operator”
[:=](https://rlang.r-lib.org/reference/quasiquotation.html#forcing-names)
to create a variable named after the “measure\_var” argument (hp in the
example). Additionally, I use the
[as\_label](https://rlang.r-lib.org/reference/as_label.html) to extract
the string value of the “measure\_var” variable.

This function is defined below and used to group by “am” (transmission
type, 0 = automatic, 1 = manual) and calculate summary statistics with
the “hp” (horsepower) variable.

``` r
car_stats <- function(groupby_var,measure_var) {
  groupby_var <- enquo(groupby_var)
  measure_var <- enquo(measure_var)
  return(mtcars %>% 
    group_by(!!groupby_var) %>%
    summarize(max=max(!!measure_var),
              min=min(!!measure_var)) %>%
          mutate(measure_var = as_label(measure_var),
            !!measure_var := NA) %>%
    select(!!groupby_var,measure_var,everything())
    )
}
hp_by_am <- car_stats(am,hp)
```

| am | measure\_var | max | min | hp |
| -: | :----------- | --: | --: | :- |
|  0 | hp           | 245 |  62 | NA |
|  1 | hp           | 335 |  52 | NA |

Now we have a flexible function for utilizing a dplyr data analysis
workflow. You can experiment with modifying this function for your own
purposes. Also, as you might suspect, you could use the same tidy
evaluation functions we used above with tidyverse packages other than
dplyr.

As an example, below I’ve defined a function that builds a scatter plot
with [ggplot2](https://ggplot2.tidyverse.org/). The function takes a
dataset and two variable names as inputs. You will notice that the data
argument needs no tidy evaluation. The
[as\_label](https://rlang.r-lib.org/reference/as_label.html) function is
this time used to extract our variable names as strings to create a plot
title with the “ggtitle” function.

``` r
library(ggplot2)
scatter_plot <- function(df,x_var,y_var) {
  x_var <- enquo(x_var)
  y_var <- enquo(y_var)
  
return(ggplot(data=df,aes(x=!!x_var,y=!!y_var)) + 
  geom_point() + theme_bw() + 
  theme(plot.title = element_text(lineheight=1, face="bold",hjust = 0.5)) +
  geom_smooth() +
  ggtitle(str_c(as_label(y_var), " vs. ",as_label(x_var)))
  )
}
scatter_plot(mtcars,disp,hp)
```

![](/rmd_images/2019-12-3-practical-tidy-evaluation/unnamed-chunk-7-1.png)<!-- -->

As you can see, we’ve plotted the “hp” (horsepower) variable against
“disp” (displacement) and added a regression line. Instead of copying
and pasting ggplot code to create the same plot with different datasets
and variables, we can just call our function.

### “Curly-Curly” and Passing Multiple Variables

To wrap things up, I’ll cover a few additional tricks for your tidy
evaluation toolbox. I will pass a list of variables using the
[syms](https://rlang.r-lib.org/reference/sym.html) function and use the
“curly-curly” {% raw %}`{{}}`{% endraw %} operator as a shortcut for
using “enquo” and “\!\!”.

Passing a list of variables allows us to group by multiple variables
using dplyr inside of a function. One quirk is that to use the syms
function we will need to pass the variable names in quotes as shown
below. Because we are passing a list, the “\!\!\!” is now used for
“groupby\_vars” instead of the “\!\!” operator we have used
previously.

I also have created new variables using a combination of a variable
value and a string with the help of the “str\_c” function from
[stringr](https://stringr.tidyverse.org/). Note that we are able to
directly evaluate the “measure\_var” argument in the summarize function
using the “curly-curly” {% raw
%}[{{}}](https://www.tidyverse.org/blog/2019/06/rlang-0-4-0/){% endraw
%} operator. We had to use a combination of “enquo” and “\!\!” to do
this before, but the “curly-curly” operator provides a shortcut for the
same operation.

Below we use this function to group by the “cyl” variable and then by
the “am” and “vs” variables (ie. the “\!\!\!” operator and “syms”
function can be used with either a list of strings or a single string)

``` r
get_stats <- function(data,groupby_vars,measure_var) {
  groupby_vars <- syms(groupby_vars)
  measure_name <- as_label(enquo(measure_var))
  return( 
    data %>% group_by(!!!groupby_vars) %>%
            summarize( !!str_c(measure_name,"_max") := max({{measure_var}}),
                       !!str_c(measure_name,"_min") := min({{measure_var}}))
    )}
cyl_hp_stats <- mtcars %>% get_stats("cyl",mpg)
gear_stats <- mtcars %>% get_stats(c("am","vs"),gear)
```

| cyl | mpg\_max | mpg\_min |
| --: | -------: | -------: |
|   4 |     33.9 |     21.4 |
|   6 |     21.4 |     17.8 |
|   8 |     19.2 |     10.4 |

| am | vs | gear\_max | gear\_min |
| -: | -: | --------: | --------: |
|  0 |  0 |         3 |         3 |
|  0 |  1 |         4 |         3 |
|  1 |  0 |         5 |         4 |
|  1 |  1 |         5 |         4 |

This concludes my introduction to the tidy evaluation framework. I hope
it is a helpful starting point for using these techniques in your code.