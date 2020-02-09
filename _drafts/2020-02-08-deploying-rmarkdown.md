---
layout: post
title:  "Deploying RMarkdown"
date:   2020-1-12
author: Jesse Cambon
tags: [ rmarkdown, data-science ]
output: 
  md_document:
    pandoc_args: ["--wrap=none"]
    variant: gfm
    preserve_yaml: TRUE
---

RMarkdown is powerful tool for creating a wide variety of documents and content on websites such as blogs. However, if you want to use RMarkdown to produce posts on a blog or Github repository, you’ll likely run into a few issues such as managing file and link paths.

While there are tools such as [blogdown](https://bookdown.org/yihui/blogdown/) to help you deploy RMarkdown on sites, this blog post describes a simple solution that I use for a jekyll blog and a GitHub repository of data science resources. This solution uses a single script file which contains RMarkdown settings adjustments and is referenced from all RMarkdown (.Rmd) files.

## The Problem

As previously mentioned, one issue you’ll run into when deploying RMarkdown on a GitHub repository or GitHub pages site is that the figure paths will no longer work when you push to your repository [as described in this blog post describes](http://www.randigriffin.com/2017/04/25/how-to-knit-for-mysite.html).

Additionally RMarkdown by default stores figure images two folder levels deep using the RMarkdown file location as its root (ie. an example path would be ./rmdname\_files/figure-gfm/image.png) I find it more conveniant for organizational purposes to place figure images in a separate root directory from my RMarkdown scripts in folders named after the RMarkdown file that generated them.

## The Solution

This solution will use a central R script to configure RMarkdown to have customizable figure image locations and to generate figure paths that will work once our content is pushed to a GitHub repository or blog. The following is the contents of this central R Script which I have named **rmd\_config.R** and placed in the root directory of my GitHub repository:

``` r
library(knitr)
library(stringr)
library(here)
# get name of file during knitting and strip file extension
rmd_filename <- str_remove(knitr::current_input(),"\\.Rmd")

# Figure path on disk = base.dir + fig.path
# Figure URL online = base.url + fig.path
knitr::opts_knit$set(base.dir = str_c(here::here(),'/'),base.url='/') # project root folder
knitr::opts_chunk$set(fig.path = str_c("rmd_images/",rmd_filename,'/'),echo=TRUE)
```

Here is what is going on in the above script:

  - The filename of our RMarkdown script is extracted using knitr and str\_remove and stored in the variable ‘rmd\_filename’.
  - The [here package](https://here.r-lib.org/) is used to establish our ‘base’ directory (the root folder of our GitHub repository). Since this could change based on what local system we are on, the here package allows us to automatically compensate for what root path we are in.
  - The ‘fig.path’, which is where our figures will be stored, is set to a folder named after the RMarkdown file being run that resides in the ‘rmd\_image’ root directory.

To utilize this above script in an RMarkdown file, we simple insert this chunk which will source the script to apply all the necessary RMarkdown settings:

``` r
library(here)
source(here::here("rmd_config.R"))
```

If you’re using RMarkdown to generate a jekyll blog post, you’ll also want to include certain YAML header content such as tags or the title of the post. To do this we can use the ‘preserve\_yaml’ setting in generating our .md file and then insert whatever yaml content we need into the header. Here is an example which will generate a github-style .md document:

``` r
---
layout: post
title:  "Deploying RMarkdown"
date:   2020-1-12
author: Jesse Cambon
tags: [ rmarkdown, data-science ]
output: 
  md_document:
    pandoc_args: ["--wrap=none"]
    variant: gfm
    preserve_yaml: TRUE
---
```

Note that the ‘pandoc\_args’ setting is to prevent pandoc from inserting extra linebreaks that don’t exist in our RMarkdown file.

## Conclusion