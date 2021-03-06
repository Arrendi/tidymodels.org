---
title: "Model tuning via grid search"
tags: [rsample, parsnip, tune, yardstick]
categories: [model tuning]
type: learn-subsection
weight: 1
description: | 
  Choose hyperparameters for a model by training on a grid of many possible parameter values.
---


  


## Introduction

To use the code in this article, you will need to install the following packages: kernlab, mlbench, and tidymodels.

This article demonstrates how to tune a model using grid search. Many models have **hyperparameters** that can't be learned directly from a single data set when training the model. Instead, we can train many models in a grid of possible hyperparameter values and see which ones turn out best. 

## Example data

To demonstrate model tuning, we'll use the Ionosphere data in the mlbench package:


```r
library(tidymodels)
library(mlbench)
data(Ionosphere)
```

From `?Ionosphere`:

> This radar data was collected by a system in Goose Bay, Labrador. This system consists of a phased array of 16 high-frequency antennas with a total transmitted power on the order of 6.4 kilowatts. See the paper for more details. The targets were free electrons in the ionosphere. "good" radar returns are those showing evidence of some type of structure in the ionosphere. "bad" returns are those that do not; their signals pass through the ionosphere.

> Received signals were processed using an autocorrelation function whose arguments are the time of a pulse and the pulse number. There were 17 pulse numbers for the Goose Bay system. Instances in this databse are described by 2 attributes per pulse number, corresponding to the complex values returned by the function resulting from the complex electromagnetic signal. See cited below for more details.

There are 43 predictors and a factor outcome. Two of the predictors are factors (`V1` and `V2`) and the rest are numeric variables that have been scaled to a range of -1 to 1. Note that the two factor predictors have sparse distributions:


```r
table(Ionosphere$V1)
#> 
#>   0   1 
#>  38 313
table(Ionosphere$V2)
#> 
#>   0 
#> 351
```

There's no point of putting `V2` into any model since is is a zero-variance predictor. `V1` is not but it _could_ be if the resampling process ends up sampling all of the same value. Is this an issue? It might be since the standard R formula infrastructure fails when there is only a single observed value:


```r
glm(Class ~ ., data = Ionosphere, family = binomial)
#> Error in `contrasts<-`(`*tmp*`, value = contr.funs[1 + isOF[nn]]): contrasts can be applied only to factors with 2 or more levels

# Surprisingly, this doesn't help: 

glm(Class ~ . - V2, data = Ionosphere, family = binomial)
#> Error in `contrasts<-`(`*tmp*`, value = contr.funs[1 + isOF[nn]]): contrasts can be applied only to factors with 2 or more levels
```

Let's remove these two problematic variables:


```r
Ionosphere <- Ionosphere %>% select(-V1, -V2)
```

## Inputs for the search

To demonstrate, we'll fit a radial basis function support vector machine to these data and tune the SVM cost parameter and the `\(\sigma\)` parameter in the kernel function:


```r
svm_mod <-
  svm_rbf(cost = tune(), rbf_sigma = tune()) %>%
  set_mode("classification") %>%
  set_engine("kernlab")
```

In this article, tuning will be demonstrated in two ways, using:

- a standard R formula, and 
- a recipe.

Let's create a simple recipe here:


```r
iono_rec <-
  recipe(Class ~ ., data = Ionosphere)  %>%
  # remove any zero variance predictors
  step_zv(all_predictors()) %>% 
  # remove any linear combinations
  step_lincomb(all_numeric())
```

The only other required item for tuning is a resampling strategy as defined by an rsample object. Let's demonstrate using basic bootstrapping:


```r
set.seed(4943)
iono_rs <- bootstraps(Ionosphere, times = 30)
```

## Optional inputs

An _optional_ step for model tuning is to specify which metrics should be computed using the out-of-sample predictions. For classification, the default is to calculate the log-likelihood statistic and overall accuracy. Instead of the defaults, the area under the ROC curve will be used. To do this, a yardstick package function can be used to create a metric set:


```r
roc_vals <- metric_set(roc_auc)
```

If no grid or parameters are provided, a set of 10 hyperparameters are created using a space-filling design (via a Latin hypercube). A grid can be given in a data frame where the parameters are in columns and parameter combinations are in rows. Here, the default will be used.

Also, a control object can be passed that specifies different aspects of the search. Here, the verbose option is turned off. 


```r
ctrl <- control_grid(verbose = FALSE)
```

## Executing with a formula

First, we can use the formula interface:


```r
set.seed(35)
formula_res <-
  svm_mod %>% 
  tune_grid(
    Class ~ .,
    resamples = iono_rs,
    metrics = roc_vals,
    control = ctrl
  )
formula_res
#> # Tuning results
#> # Bootstrap sampling 
#> # A tibble: 30 x 4
#>    splits            id          .metrics          .notes          
#>    <list>            <chr>       <list>            <list>          
#>  1 <split [351/120]> Bootstrap01 <tibble [10 × 6]> <tibble [0 × 1]>
#>  2 <split [351/130]> Bootstrap02 <tibble [10 × 6]> <tibble [0 × 1]>
#>  3 <split [351/137]> Bootstrap03 <tibble [10 × 6]> <tibble [0 × 1]>
#>  4 <split [351/141]> Bootstrap04 <tibble [10 × 6]> <tibble [0 × 1]>
#>  5 <split [351/131]> Bootstrap05 <tibble [10 × 6]> <tibble [0 × 1]>
#>  6 <split [351/131]> Bootstrap06 <tibble [10 × 6]> <tibble [0 × 1]>
#>  7 <split [351/127]> Bootstrap07 <tibble [10 × 6]> <tibble [0 × 1]>
#>  8 <split [351/123]> Bootstrap08 <tibble [10 × 6]> <tibble [0 × 1]>
#>  9 <split [351/131]> Bootstrap09 <tibble [10 × 6]> <tibble [0 × 1]>
#> 10 <split [351/117]> Bootstrap10 <tibble [10 × 6]> <tibble [0 × 1]>
#> # … with 20 more rows
```

The `.metrics` column contains tibbles of the performance metrics for each tuning parameter combination:


```r
formula_res %>% 
  select(.metrics) %>% 
  slice(1) %>% 
  pull(1)
#> [[1]]
#> # A tibble: 10 x 6
#>        cost rbf_sigma .metric .estimator .estimate .config
#>       <dbl>     <dbl> <chr>   <chr>          <dbl> <chr>  
#>  1  0.00849  1.11e-10 roc_auc binary         0.815 Model01
#>  2  0.176    7.28e- 8 roc_auc binary         0.839 Model02
#>  3 14.9      3.93e- 4 roc_auc binary         0.870 Model03
#>  4  5.51     2.10e- 3 roc_auc binary         0.919 Model04
#>  5  1.87     3.53e- 7 roc_auc binary         0.838 Model05
#>  6  0.00719  1.45e- 5 roc_auc binary         0.832 Model06
#>  7  0.00114  8.41e- 2 roc_auc binary         0.969 Model07
#>  8  0.950    1.74e- 1 roc_auc binary         0.984 Model08
#>  9  0.189    3.13e- 6 roc_auc binary         0.832 Model09
#> 10  0.0364   4.96e- 9 roc_auc binary         0.839 Model10
```

To get the final resampling estimates, the `collect_metrics()` function can be used on the grid object:


```r
estimates <- collect_metrics(formula_res)
estimates
#> # A tibble: 10 x 8
#>        cost rbf_sigma .metric .estimator  mean     n std_err .config
#>       <dbl>     <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>  
#>  1  0.00849  1.11e-10 roc_auc binary     0.822    30 0.00718 Model01
#>  2  0.176    7.28e- 8 roc_auc binary     0.871    30 0.00525 Model02
#>  3 14.9      3.93e- 4 roc_auc binary     0.916    30 0.00497 Model03
#>  4  5.51     2.10e- 3 roc_auc binary     0.960    30 0.00378 Model04
#>  5  1.87     3.53e- 7 roc_auc binary     0.871    30 0.00524 Model05
#>  6  0.00719  1.45e- 5 roc_auc binary     0.871    30 0.00534 Model06
#>  7  0.00114  8.41e- 2 roc_auc binary     0.966    30 0.00301 Model07
#>  8  0.950    1.74e- 1 roc_auc binary     0.979    30 0.00204 Model08
#>  9  0.189    3.13e- 6 roc_auc binary     0.871    30 0.00536 Model09
#> 10  0.0364   4.96e- 9 roc_auc binary     0.871    30 0.00537 Model10
```

The top combinations are:


```r
show_best(formula_res, metric = "roc_auc")
#> # A tibble: 5 x 8
#>       cost rbf_sigma .metric .estimator  mean     n std_err .config
#>      <dbl>     <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>  
#> 1  0.950   0.174     roc_auc binary     0.979    30 0.00204 Model08
#> 2  0.00114 0.0841    roc_auc binary     0.966    30 0.00301 Model07
#> 3  5.51    0.00210   roc_auc binary     0.960    30 0.00378 Model04
#> 4 14.9     0.000393  roc_auc binary     0.916    30 0.00497 Model03
#> 5  0.00719 0.0000145 roc_auc binary     0.871    30 0.00534 Model06
```

##  Executing with a recipe

Next, we can use the same syntax but pass a *recipe* in as the pre-processor argument:


```r
set.seed(325)
recipe_res <-
  svm_mod %>% 
  tune_grid(
    iono_rec,
    resamples = iono_rs,
    metrics = roc_vals,
    control = ctrl
  )
recipe_res
#> # Tuning results
#> # Bootstrap sampling 
#> # A tibble: 30 x 4
#>    splits            id          .metrics          .notes          
#>    <list>            <chr>       <list>            <list>          
#>  1 <split [351/120]> Bootstrap01 <tibble [10 × 6]> <tibble [0 × 1]>
#>  2 <split [351/130]> Bootstrap02 <tibble [10 × 6]> <tibble [0 × 1]>
#>  3 <split [351/137]> Bootstrap03 <tibble [10 × 6]> <tibble [0 × 1]>
#>  4 <split [351/141]> Bootstrap04 <tibble [10 × 6]> <tibble [0 × 1]>
#>  5 <split [351/131]> Bootstrap05 <tibble [10 × 6]> <tibble [0 × 1]>
#>  6 <split [351/131]> Bootstrap06 <tibble [10 × 6]> <tibble [0 × 1]>
#>  7 <split [351/127]> Bootstrap07 <tibble [10 × 6]> <tibble [0 × 1]>
#>  8 <split [351/123]> Bootstrap08 <tibble [10 × 6]> <tibble [0 × 1]>
#>  9 <split [351/131]> Bootstrap09 <tibble [10 × 6]> <tibble [0 × 1]>
#> 10 <split [351/117]> Bootstrap10 <tibble [10 × 6]> <tibble [0 × 1]>
#> # … with 20 more rows
```

The best setting here is:


```r
show_best(recipe_res, metric = "roc_auc")
#> # A tibble: 5 x 8
#>      cost rbf_sigma .metric .estimator  mean     n std_err .config
#>     <dbl>     <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>  
#> 1 15.6    0.182     roc_auc binary     0.981    30 0.00213 Model04
#> 2  0.385  0.0276    roc_auc binary     0.978    30 0.00222 Model03
#> 3  0.143  0.00243   roc_auc binary     0.930    30 0.00443 Model06
#> 4  0.841  0.000691  roc_auc binary     0.892    30 0.00504 Model07
#> 5  0.0499 0.0000335 roc_auc binary     0.872    30 0.00521 Model08
```



## Session information


```
#> ─ Session info ───────────────────────────────────────────────────────────────
#>  setting  value                       
#>  version  R version 4.0.2 (2020-06-22)
#>  os       macOS Catalina 10.15.6      
#>  system   x86_64, darwin17.0          
#>  ui       X11                         
#>  language (EN)                        
#>  collate  en_US.UTF-8                 
#>  ctype    en_US.UTF-8                 
#>  tz       America/Denver              
#>  date     2020-07-21                  
#> 
#> ─ Packages ───────────────────────────────────────────────────────────────────
#>  package    * version date       lib source        
#>  broom      * 0.7.0   2020-07-09 [1] CRAN (R 4.0.0)
#>  dials      * 0.0.8   2020-07-08 [1] CRAN (R 4.0.0)
#>  dplyr      * 1.0.0   2020-05-29 [1] CRAN (R 4.0.0)
#>  ggplot2    * 3.3.2   2020-06-19 [1] CRAN (R 4.0.0)
#>  infer      * 0.5.3   2020-07-14 [1] CRAN (R 4.0.2)
#>  kernlab    * 0.9-29  2019-11-12 [1] CRAN (R 4.0.2)
#>  mlbench    * 2.1-1   2012-07-10 [1] CRAN (R 4.0.2)
#>  parsnip    * 0.1.2   2020-07-03 [1] CRAN (R 4.0.1)
#>  purrr      * 0.3.4   2020-04-17 [1] CRAN (R 4.0.0)
#>  recipes    * 0.1.13  2020-06-23 [1] CRAN (R 4.0.0)
#>  rlang        0.4.7   2020-07-09 [1] CRAN (R 4.0.2)
#>  rsample    * 0.0.7   2020-06-04 [1] CRAN (R 4.0.0)
#>  tibble     * 3.0.3   2020-07-10 [1] CRAN (R 4.0.2)
#>  tidymodels * 0.1.1   2020-07-14 [1] CRAN (R 4.0.2)
#>  tune       * 0.1.1   2020-07-08 [1] CRAN (R 4.0.0)
#>  workflows  * 0.1.2   2020-07-07 [1] CRAN (R 4.0.0)
#>  yardstick  * 0.0.7   2020-07-13 [1] CRAN (R 4.0.2)
#> 
#> [1] /Library/Frameworks/R.framework/Versions/4.0/Resources/library
```
