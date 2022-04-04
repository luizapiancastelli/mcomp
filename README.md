Multivariate COM-Poisson distribution: Sarmanov method and Bayesian
inference
================

Before installing `multcp` ensure that C++ compiler is properly set up
to be used with R. We suggest to test this with `library(RcppArmadillo)`
as our package links to this framework. Once this is done, `multcp` can
be installed from Git Hub with the aid of `devtools` and is ready to be
used.

``` r
library(devtools)
install_github("luizapiancastelli/multcp")
library(multcp)
```

### 1\. Simulating MCOMP data

Data can be simulated using the MCOMP sampler algorithm (Section ?) with
the `rmcomp` functions, taking in the number of random draws `n`,
parameter vectors `lambda` and `nu` of lenght `d`, \(d \times d\) matrix
\(\boldsymbol{\underline{\delta}}\) and scalar `omega`. `N_r` is the
number of auxiliary draws used to estimate the intractable ratios and
the conditional probability function truncation tolerance can be changed
by supplying the argument `tol`.

``` r
n = 100
lambda = c(1.5, 1, 0.5)
nu = c(1, 0.5, 2)
omega = 1
delta = matrix(0, ncol = length(lambda), nrow = length(lambda))
delta[1,2] = 2.5
delta[1,3] = 2
delta[2,3] = 3
N_r = 10000

Y = rmcomp(n, lambda, nu, delta, omega, N_r)
colMeans(Y)
```

    ## [1] 1.52 1.29 0.37

``` r
cor(Y)
```

    ##            [,1]        [,2]        [,3]
    ## [1,] 1.00000000 0.377320776 0.051008683
    ## [2,] 0.37732078 1.000000000 0.003932739
    ## [3,] 0.05100868 0.003932739 1.000000000

### 2\. GIMH

Pseudo-marginal inference is available via the `GIMH` function that runs
in parallel `chains` of `n_iter` iterations with `burn_in` period using
`ncores`. Auxiliary draws calibration is done via the arguments `N_r`
(integer) and `N_z` that can be an integer or a list containing the
target log-likelihood standard deviation `target_sd` calibrating `N_z`
with initialisation `init`. Prior parameters are supplied via a
`priors_list`, and parallelization can be disabled by setting
`ncores=1`. The following example uses the manuscript’s simulated data
set of Configuration 1, with the other settings available with names
`Y_c2, Y_c3, Y_c4`.

``` r
data("Y_c1") #Loads the simulated data of configuration 1

N_aux_r = 10000
N_aux_z = list('target_sd' =1.2, 'init' = 10000)
burn_in = 10000
n_iter = 2000

priors_list = list('lambda' = list('sd' = 100),
                   'nu' = list('sd' = 100),
                   'omega' = list('sd' = 5))

run_gimh = GIMH(Y_c1, burn_in, n_iter, N_aux_r, N_aux_z, priors_list, chains = 2, ncores = 2)
names(run_gimh) #Returns a list with timing and raw MCMC object
```

The function returns a list with the total algorithm’s runtime `$time`
and a raw MCMC object `$mcmc` of length `chains`. The former contains
the posterior draws, acceptance rates, initial values and proposal
parameters and can be processed and summarised using `process_mcmc` and
`posterior_summaries`.

``` r
names(run_gimh$mcmc[[1]]) #Output available for each of the {1,...,chains}

process_gimh = process_mcmc(run_gimh$mcmc)
gimh_summary = posterior_summaries(run_gimh$mcmc)

gimh_summary$Rhat     #Rhat statistic
gimh_summary$Stats    #Some posterior summaries using the combined MCMC draws

gridExtra::grid.arrange(gimh_summary$Density_plots, gimh_summary$Trace_plots, ncol =2)
```

### 3\. Noisy Exchange algorithm

The `Exchange` function works similarly, but `tol` (\(\epsilon\)) can be
supplied in the place of \(N_z\) that is no longer used. Output is in
the same format and the functions `process_mcmc` and
`posterior_summaries` also
apply.

``` r
run_exchange = Exchange(n_iter, burn_in, Y_c1, N_aux_r, priors_list, chains=2,  ncores = 2)

exchange_summary = posterior_summaries(run_exchange$mcmc)
exchange_summary$Stats
```

### 4\. Premier League Regression

``` r
data("Y_premier")
Y = as.matrix(Y_premier[,1:2])
X =Y_premier[,3]

burn_in = 10000
n_iter = 2000
N_aux_r = 10000

priors_list = list('gamma0' = list('sd' = 100), 
                   'gamma' = list('sd' = 100),
                   'nu' = list('sd' = 100),
                   'omega' = list('sd' = 5))

run_premier_exchange = Exchange_regression(burn_in, n_iter, Y, X, N_aux_r, priors_list)
```