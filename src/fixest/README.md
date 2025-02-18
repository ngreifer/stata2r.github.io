---
title: fixest
---

# Regression analysis with fixest

[**fixest**](https://lrberge.github.io/fixest) (by Laurent Bergé) is a package 
designed from the ground up in C++ to make running regressions fast and 
incredibly easy. It provides in-built support for a variety of linear and 
non-linear models, as well as regression tables and plotting methods. 

## Installation

Before continuing, make sure that you have installed **fixest**. You only 
have to do this once (or as often as you want to update the package).

<div class="code--container grid-cols-1">
<div>

```r
# Install from CRAN
install.packages("fixest")

# Alternatively, you can install the latest development version
# install.packages("fixest", repos = "https://fastverse.r-universe.dev")
```
</div>
</div>

Once **fixest** is installed, don't forget to load it whenever you want to 
use it. Unlike Stata, you have to re-load a package every time you start a new R 
session.

<div class="code--container grid-cols-1">
<div>

```r
# Load fixest into our current R session
library(fixest)
```
</div>
</div>

All of the examples in this section will use a modified dataset from the CPS
with some added variables for demonstration purposes. To load the data run the
following:

<div class="code--container">
<div>

```stata
import delimited using ///
    "https://raw.githubusercontent.com/stata2r/stata2r.github.io/main/data/cps_long.csv", clear
```
</div>
<div>

```r
# Base R reads CSVs too, but we'll use data.table here
dat = data.table::fread('https://raw.githubusercontent.com/stata2r/stata2r.github.io/main/data/cps_long.csv')
```
</div>
</div>


## Introduction

The [**fixest**](https://lrberge.github.io/fixest/index.html) package contains a highly flexible set of tools that allow you to estimate a fairly large set of standard regression models. While the package certainly doesn't cover *every* model that exists, there is a non-negligible subset of Stata users for whom every model they've ever needed to run is covered by **fixest**.

This includes regular ol' linear regression in the `feols()` function, which builds off of the Base R standard regression function `lm(),` but also includes things like instrumental variables via 2SLS, and of course support for as many fixed effects as you'd like. **fixest** isn't limited to linear regression either, covering IV and fixed-effects support for a wide range of GLM models like logit, probit, Poisson, negative binomial, and so on in `feglm()` and `fepois().`

**fixest** covers all of this while being _very_ fast. If you felt a speed boost going from Stata's `xtreg` to `reghdfe,` get ready for another significant improvement when moving to **fixest**.

You also get a fair amount of convenience. Adjusting your standard errors to be heteroskedasticity-robust or clustered can be a pain in other R regression functions, but it is easy in **fixest** with the `vcov` option. Regression tables, coefficient and interaction-margin plots, selecting long lists of controls without having to type them all in, lagged variables, retrieving estimated fixed effects, Wald tests, and the choice of reference for categorical variables are all made easy. You even get some stuff that's rather tricky in Stata, like automatically iterating over a bunch of model specifications, basic and staggered difference-in-difference support, or Conley standard errors.

Using **fixest** for regression starts with writing a formula. While there are plenty of bells and whistles to add, at its core regression formulas take the form **`y ~ x1 + x2 | fe1 + fe2`**, where `y` is the outcome, `x1` and `x2` are predictors, and `fe1` and `fe2` are your sets of fixed effects.

                     
## Models

Unlike Stata, which only ever has one active dataset in memory, remember that
having multiple datasets in your global environment is the norm in R. We
highlight this difference to head off a very common error for new Stata R users:
_you need to specify which dataset you're using in your model calls_, e.g.
`feols(..., data = dat)`. We'll see lots of examples below. At the same time,
note that **fixest** allows you to set various [global
options](https://lrberge.github.io/fixest/reference/index.html#section-default-values),
including which dataset you want to use for all of your regressions. Again,
we'll see examples below.

           
### Simple model

<div class='code--container'>
<div>

```stata
reg wage educ 
reg wage educ age
```
</div>
<div>

```r
feols(wage ~ educ, data = dat) 
feols(wage ~ educ + age, data = dat)

# Aside 1: `data = ...` is always the first argument 
# after the model formula. So many R users would just 
# write: 
feols(wage ~ educ, dat) 

# Aside 2: You can also set your dataset globally so 
# that you don't have to reference it each time. 
setFixest_estimation(data = dat) 
feols(wage ~ educ) 
feols(wage ~ educ + age) 
# etc.
```
</div>
</div>
           
### Categorical variables

<div class='code--container'>
<div>

```stata
reg wage educ i.treat 

* Specifying a baseline:
reg wage educ ib1.treat
```
</div>
<div>

```r
feols(wage ~ educ + i(treat), dat) 

# Specifying a baseline:
feols(wage ~ educ + i(treat, ref = 1), dat)
```
</div>
</div>
           
### Fixed effects

<div class='code--container'>
<div>

```stata
reghdfe wage educ, absorb(countyfips) cluster(countyfips) 





reghdfe wage educ, absorb(countyfips)  

* Add more fixed effects... 
reghdfe wage educ, absorb(countyfips year) ///
                   vce(cluster countyfips year) 
reghdfe wage educ, absorb(countyfips#year) /// 
                   vce(cluster countyfips#year)
```
</div>
<div>

```r
feols(wage ~ educ | countyfips, dat) 

# Aside: fixest automatically clusters SEs by the first 
# fixed effect (if there are any). We'll get to SEs 
# later, but if you just want iid errors for a fixed 
# effect model: 
feols(wage ~ educ | countyfips, dat, vcov = 'iid') 

# Add more fixed effects... 
feols(wage ~ educ | countyfips + year, 
      dat, vcov = ~countyfips + year) 
feols(wage ~ educ | countyfips^year, 
      dat) # defaults to vcov = ~countyfips^year
```
</div>
</div>
           
### Instrumental variables

<div class='code--container'>
<div>

```stata
ivreg 2sls wage (educ = age) 
ivreg 2sls wage marr (educ = age) 

* With fixed effects 
ivreghdfe 2sls wage marr (educ = age), absorb(countyfips)
```
</div>
<div>

```r
feols(wage ~ 1 | educ ~ age, dat)  
feols(wage ~ marr | educ ~ age, dat) 

# With fixed effects (IV 1st stage always comes last) 
feols(wage ~ marr | countyfips | educ ~ age, dat)
```
</div>
</div>
           
### Nonlinear models

While we don't really show it here, note that (almost) all of the functionality that
this page demonstrates w.r.t. `feols()` carries over to **fixest's** non-linear 
estimation functions too (`feglm()`, `fepois()`, etc.). This includes SE 
adjustment, and so forth.

<div class='code--container'>
<div>

```stata
xtset statefips
logit marr age black hisp

* Note: Attempting to replicate the feglm() model with fixed
* effects at right using xtlogit or xtprobit leads to 
* numerical overflow or matsize issues


ppmlhdfe educ age black hisp, absorb(statefips year) ///
	                      vce(cluster statefips)
```
</div>
<div>

```r
feglm(marr ~ age + black + hisp, 
      dat, family = 'logit')

# Add fixed effects (probit this time)
feglm(marr ~ age + black + hisp | statefips + year, 
      dat, family = 'probit')

# fepois() is there for Poisson regression
fepois(educ ~ age + black + hisp | statefips + year, dat)
```
</div>
</div>	  


### Macros, wildcards and shortcuts

<div class='code--container'>
<div>

```stata
local ctrls age black hisp marr 
reg wage educ `ctrls' 

reg wage educ x* 
reg wage educ *sp  
reg wage educ *ac*
```
</div>
<div>

```r
ctrls = c("age", "black", "hisp", "marr") 
feols(wage ~ educ + .[ctrls], dat) 

feols(wage ~ educ + ..('^x'), dat) # ^ = starts with 
feols(wage ~ educ + ..('sp$'), dat) # $ = ends with 
feols(wage ~ educ + ..('ac'), dat) 

# Many more macro options. See `?setFixest_fml` and
# `?setFixest_estimation`. Example (reminder) where 
# you set your dataset globally, so you don't have to 
# retype `data = ...` anymore. 
setFixest_estimation(data = dat) 
feols(wage ~ educ) 
feols(wage ~ educ + .[ctrls] | statefips) 
# Etc.
```
</div>
</div>
          

### Multi-model estimations (advanced)

**fixest** supports a variety of
[multi-model](https://lrberge.github.io/fixest/articles/multiple_estimations.html) 
capabilities. Not only are these efficient from a coding perspective (you can get 
away with much less typing), but they are also highly optimized. For example, 
if you run a multi-model estimation with the same group of fixed-effects then 
**fixest** will only compute those fixed-effects _once_ for all models. The next
group of examples are meant to highlight some specific examples of this
functionality. They don't necessarily have direct Stata equivalents that we are 
aware of. Moreover, while we don't show it here, please note that all of these
options can be combined (e.g. split sample with stepwise regression). 
Multi-model objects can also be sent directly to [presentation](#presentation) 
functions like `etable()` and `coefplot()`.

#### Split sample

```r
## Separate regressions for hispanic and non-hispanic
feols(wage ~ educ | countyfips, dat, split = ~hisp)

# As above, but now includes the full sample as a 3rd reg
feols(wage ~ educ | countyfips, dat, fsplit = ~hisp)
```

#### Multiple dependent variables

```r
# Regress wage & educ separately on the same dep. vars & FEs
feols(c(wage, educ) ~ age + marr | countyfips + year, dat) 
```

#### Stepwise regression

```r
# Stepwise. First reg doesn't include "marr", second reg
# doesn't include "age"
feols(wage ~ educ + sw(age, marr), dat) 

# Cumulative stepwise. As above, except now the second reg
# includes "age".
feols(wage ~ educ + csw(age, marr), dat)

# Stepwise operators work in the FE slot too
feols(wage ~ educ | csw(year, statefips), dat)

# Aside: You have also use "sw0()" and "csw0()", in which 
# case you'll get an extra regression at the start that 
# doesn't include the stepwise components. 
```


## Interactions

           
### Interact continuous variables

<div class='code--container'>
<div>

```stata
reg wage c.educ#c.age 
reg wage c.educ##c.age 

* Polynomials 
reg wage c.age#c.age 
reg wage c.age##c.age 
```
</div>
<div>

```r
feols(wage ~ educ:age, dat) 
feols(wage ~ educ*age, dat) 

# Polynomials 
feols(wage ~ I(age^2), dat) 
feols(wage ~ poly(age, 2, raw = TRUE))
```
</div>
</div>
           
### Interact categorical variables

<div class='code--container'>
<div>

```stata
reg wage i.treat#i.hisp 




reg wage i.treat i.treat#i.hisp
reg wage i.treat##i.hisp
```
</div>
<div>

```r
feols(wage ~ i(treat, i.hisp), dat) 

# Aside: i() is a fixest-specific shortcut that also 
# has synergies with some other fixest functions. But 
# base R interaction operators all still work, e.g. 
feols(wage ~ factor(treat)/factor(hisp), dat) 
feols(wage ~ factor(treat)*factor(hisp), dat)
```
</div>
</div>
           
### Interact categorical with continuous variables

<div class='code--container'>
<div>

```stata
reg wage i.treat#c.age 




reg wage i.treat#c.age 
reg wage i.treat i.treat#c.age 
reg wage i.treat##c.age
```
</div>
<div>

```r
feols(wage ~ i(treat, age), dat) 

# Aside: i() is a fixest-specific shortcut that also 
# has synergies with some other fixest functions. But 
# base R interaction operators all still work, e.g. 
feols(wage ~ factor(treat):age, dat) 
feols(wage ~ factor(treat)/age, dat) 
feols(wage ~ factor(treat)*age, dat)
```
</div>
</div>

### Difference-in-differences

In addition to the ability to estimate a difference-in-differences design using
two-way fixed effects (if the design is appropriate for that; no staggered
treatment, for instance), **fixest** offers several other DID-specific tools.
The below examples use generic data sets, since the CPS data used in the rest of
this page is not appropriate for DID.

<div class='code--container'>
<div>

```stata
* No immediate Stata equivalent to did_means that we know of,
* although you could replicate much of it by hand with an 
* elaborate call to table

* Sun and Abraham can be estimated using the 
* eventstudyinteract package on ssc
```
</div>
<div>

```r
# did_means() provides tables of means, SEs, and treatment/
# control and pre/post differences for 2x2 DID
did_means(outcome + control ~ treat | post)

# sunab() produces interactions of the type that allow you to
# estimate the Sun & Abraham model for staggered treatment 
# timing, and automatically get average treatment effects for
# each relative period
feols(y ~ control + sunab(year_treated, year))
```
</div>
</div>	  
           
### Interact fixed effects

<div class='code--container'>
<div>

```stata
* Combine fixed effects 
reghdfe wage educ, absorb(statefips#year) 

* Varying slopes (e.g. time trend for each state) 
reghdfe wage educ, absorb(statefips#c.year) ///
	           vce(cluster statefips#c.year)
```
</div>
<div>

```r
# Combine fixed effects 
feols(wage ~ educ | statefips^year, dat) 

# Varying slopes (e.g. time trend for each state) 
feols(wage ~ educ | statefips[year], dat)
```
</div>
</div>
                     
                     
## Standard errors

While you can specify standard errors inside the original **fixest** model call
(just like Stata), a unique feature of R is that you can adjust errors for an
existing model _on-the-fly_. This has [several
benefits](https://grantmcdermott.com/better-way-adjust-SEs), including being
much more efficient since you don't have to re-estimate your whole model. We'll
try to highlight examples of both approaches below.

           
### HC

<div class='code--container'>
<div>

```stata
reg wage educ, vce(robust) 
reg wage educ, vce(hc3)
```
</div>
<div>

```r
feols(wage ~ educ, dat, vcov = 'hc1') 
feols(wage ~ educ, dat, vcov = sandwich::vcovHC) 

# Note: You can also adjust the SEs of an existing model 
m = feols(wage ~ educ, dat) # iid
summary(m, vcov = 'hc1')    # switch to HC1
```
</div>
</div>
           
### HAC

<div class='code--container'>
<div>

```stata
xtset id year
ivreghdfe wage educ, bw(auto) vce(robust)
```
</div>
<div>

```r
feols(y ~ x, dat, vcov = 'NW', panel.id = ~unit + time)
# feols(y ~ x, dat, vcov = 'NW') # if panel id already set (see below)
```
</div>
</div>
           
### Clustered

<div class='code--container'>
<div>

```stata
reghdfe wage educ, absorb(countyfips) /// 
                   vce(cluster countyfips) 

* Twoway clustering etc. 
reghdfe wage educ, absorb(countyfips year) ///
                   vce(cluster countyfips year) 



reghdfe wage educ, absorb(countyfips#year) ///
                   vce(cluster countyfips#year)
```
</div>
<div>

```r
feols(wage ~ educ | countyfips, dat) # Auto clusters by FE 
# feols(wage ~ educ | countyfips, dat, vcov = ~countyfips) # ofc can be explicit too 

# Twoway clustering etc. 
feols(wage ~ educ | countyfips + year, 
      dat, vcov = ~countyfips + year) 
# feols(wage ~ educ | countyfips + year, 
#      dat, vcov = 'twoway') ## same as above

feols(wage ~ educ | countyfips^year, 
      dat, vcov = ~countyfips^year) 
```
</div>
</div>
           
### Conley standard errors

<div class='code--container'>
<div>

```stata
* See: http://www.trfetzer.com/conley-spatial-hac-errors-with-fixed-effects/
```
</div>
<div>

```r
feols(wage ~ educ, dat, vcov = conley("25 mi"))
```
</div>
</div>

### On-the-fly SE adjustment

We're belabouring the point now, but one last reminder that you can adjust the
standard errors for existing models "on the fly" by passing the `vcov = ...`
argument. There's no performance penalty, since the adjustment is done 
instantaneously and it therefore has the virtue of separating the mechanical
_computation_ stage of model estimation from the _inference_ stage. As we'll see 
below, on-the-fly SE adjustment works for a variety of other **fixest** 
functions, e.g. `etable()`. But here is a quick example using `summary()`:

```r
m = feols(wage ~ educ | countyfips + year, dat) 
m                                    # Clustered by countyfips (default)
summary(m, vcov = 'iid')             # Switch to iid errors
summary(m, vcov = 'twoway')          # Cluster by countyfips and year 
summary(m, vcov = ~countyfips^year)  # Cluster by countyfips*year interaction
```
              
## Presentation

           
### Regression table

<div class='code--container'>
<div>

```stata
reg wage educ age 
eststo est1 
esttab est1

* Add second regression
reg wage educ age black hisp
eststo est2
esttab est1 est2

* Export to TeX
esttab using "regtable.tex", replace 
```
</div>
<div>

```r
est1 = feols(wage ~ educ + age, dat) 
etable(est1)


# Add second regression
est2 = feols(wage ~ educ + age + black + hisp, dat) 
etable(est1, est2)


# Export to Tex
etable(est1, est2, file = "regtable.tex")
```
</div>
</div>

**Note:** The `etable()` function is extremely flexible and includes support for
many things that we won't show you here. See the relevant vignettes for more
([1](https://lrberge.github.io/fixest/articles/exporting_tables.html),
[2](https://lrberge.github.io/fixest/articles/etable_new_features.html)). Below we highlight a few unique features that don't have direct Stata
equivalents. (You could potentially mimic with a loop, but that will require 
more code and be slower, since your whole model has to be re-estimated each 
time.)

```r
# SEs for existing models can be adjusted on-the-fly 
etable(est1, est2, vcov = 'hc1') 

# Report multiple SEs for the same model 
etable(est1, vcov = list('iid', 'hc1', ~id, ~countyfips)) 

# Multi-model example: Two dep. vars, stepwise coefs, 
# varying slopes, split sample, etc. (18 models in total!)
est_mult = feols(c(wage, age) ~ educ + csw0(hisp, black) | 
                     statefips[year], 
                 dat, fsplit = ~marr) 
etable(est_mult, vcov = ~statefips^year)
```

### Joint test of coefficients

<div class='code--container'>
<div>

```stata
* Rename so we can use the wildcard later
rename (black hisp) (raceeth_black raceeth_hisp)
regress wage educ age raceeth_* marr 
testparm raceeth_black raceeth_hisp
testparm raceeth_*
```
</div>
<div>

```r
# Rename so we can use a regular expression later
data.table::setnames(dat, c('black','hisp'), c('raceeth_black','raceeth_hisp'))
est1 = feols(wage ~ educ + age + ..('raceeth_') + marr, dat)
wald(est1, c('raceeth_black','raceeth_hisp'))
wald(est1, 'raceeth_')
```
</div>
</div>
           
### Coefficient plot

<div class='code--container'>
<div>

```stata
* Assume we have est1 and est2 from above 
coefplot est1 
coefplot est1 est2
```
</div>
<div>

```r
# Assume we have est1 and est2 from above 
coefplot(est1) 
coefplot(list(est1, est2))
```
</div>
</div>
                     
### Interaction Plot

<div class='code--container'>
<div>

```stata
regress wage hisp##c.age

* Show how effect differs by group
margins hisp, dydx(age)
marginsplot

# Show predictive margins with an interaction
regress wage hisp##c.age
margins hisp, at(age = (16(1)55))
* The recast here gives a line and error area instead of points and lines
marginsplot, recast(line) recastci(rarea)
```
</div>
<div>

```r
est1 = feols(wage ~ i(hisp, age), dat) 

# Show how effect differs by group
iplot(est1)


# Show predictive margins with an interaction
# This requires plot_cap from the marginaleffects package
library(marginaleffects)
plot_cap(est1, condition = c('age','hisp'))
```
</div>
</div>        
		
## Panel

**Note:** You don't need to specify your panel variables globally and this functionality is mostly for convenience features associated with time-series operations like leads and lags. You can also use `panel(dat, ~ id + var)` to do so on-the-fly in your regression call. But Laurent, the **fixest** author, recommends setting the panel ID globally when applicable, so that's what we do below.

           
### Lag variables

<div class='code--container'>
<div>

```stata
xtset id year 
reg wage educ l1.wage
```
</div>
<div>

```r
setFixest_estimation(panel.id = ~id+year)
feols(wage ~ educ + l(wage, 1), dat)
# feols(wage ~ educ + l(wage, 1), dat, panel.id = ~id+year) # if not set
```
</div>
</div>
           
### Lead variables

<div class='code--container'>
<div>

```stata
xtset id year 
reg wage educ f1.wage
```
</div>
<div>

```r
# setFixest_estimation(panel.id = ~id+year) # already set
feols(wage ~ educ + f(wage, 1), dat)
```
</div>
</div>
           
### First difference

<div class='code--container'>
<div>

```stata
xtset id year 
reg wage educ D.x
```
</div>
<div>

```r
# setFixest_estimation(panel.id = ~id+year) # already set
feols(wage ~ educ + d(wage), dat)
```
</div>
</div>
