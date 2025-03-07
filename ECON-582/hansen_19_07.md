Hansen Exercise 19.7
================
Lukas Hager
5/27/2021

### Clear Environment and Import Packages

``` r
rm(list = ls())
gc()
```

    ##          used (Mb) gc trigger (Mb) limit (Mb) max used (Mb)
    ## Ncells 416913 22.3     862270 46.1         NA   657820 35.2
    ## Vcells 791542  6.1    8388608 64.0      16384  1802519 13.8

``` r
gc()
```

    ##          used (Mb) gc trigger (Mb) limit (Mb) max used (Mb)
    ## Ncells 417368 22.3     862270 46.1         NA   657820 35.2
    ## Vcells 792537  6.1    8388608 64.0      16384  1802519 13.8

``` r
library(openxlsx)
library(data.table)
library(broom)
library(furrr)
library(tidyverse)
```

### Read In Data and Filter to Relevant Observations

``` r
data_og <- 'https://www.ssc.wisc.edu/~bhansen/econometrics/DDK2011.xlsx' %>% 
  read.xlsx(., na.strings = '.')

reg_data <- data_og %>% 
  filter(!is.na(totalscore), !is.na(percentile), tracking == 1, girl == 0) %>% 
  select(totalscore, perc = percentile, schoolid) %>% 
  data.table()

cat(str_interp('We are left with ${format(nrow(reg_data), big.mark = ",")} observations.'))
```

We are left with 1,473 observations.

### Calculate Rule-of-Thumb (ROT) Bandwidth

We will use the Fan and Gijbels (1996) suggestion that `q=4`, so we will
fit a fourth-order polynomial.

``` r
reg_data_poly <- reg_data %>% 
  mutate(perc_2 = perc^2,
         perc_3 = perc^3,
         perc_4 = perc^4) %>% 
  data.table()

poly_reg <- lm(totalscore ~ perc + perc_2 + perc_3 + perc_4, 
               data = reg_data_poly) 

poly_reg_2d <- poly_reg %>% 
  tidy() %>% 
  filter(str_detect(term, 'perc\\_[2-4]')) %>% 
  select(term, estimate) %>% 
  mutate(power = as.numeric(str_extract(term, '\\d')),
         estimate_adj = estimate * power * (power - 1))

fitted_2d <- reg_data_poly %>% 
  rowwise() %>% 
  mutate(fit = sum(c(1, perc, perc_2) * poly_reg_2d$estimate_adj)) %>% 
  ungroup()

B_hat <- 1 / nrow(reg_data) * sum(((1 / 2) * fitted_2d$fit)^2)

support_diff <- max(reg_data$perc) - min(reg_data$perc)

sigma_2 <- var(poly_reg$residuals) * support_diff

h_rot <- .58 * (sigma_2 / (nrow(reg_data) * B_hat))^(1/5)

cat(str_interp('We get a ROT bandwidth of `h_rot = ${round(h_rot, 2)}`.'))
```

We get a ROT bandwidth of `h_rot = 5.34`.

### Calculate Conventional Cross-Validation Bandwidth

To do this, we need to define a grid of `h` values. We will iterate over
the grid and select the value that minimizes the cross-validation loss,
which is defined as the sum of the leave-one-out estimator errors. This
is pretty time-intensive (maybe because my code isn’t efficient) because
each value of `h` that we try has \\(n^2\\) calculations for \\(\\) and
the error.

As a methodological note: I compute the min by running the algorithm on
a coarse grid from 4 to 20 in increments of 1 to match the bounds in
Figure 19.6(a), and then on a finer grid in increments of .2 in a
neighborhood of radius 1 around the minimum found on the coarse grid to
get the final value.

``` r
h_grid_coarse <- seq(4, 20, by = 1)

get_ll_beta <- function(val,x,h){
  Z <- reg_data[-val] %>% 
    select(perc) %>% 
    mutate(cons = 1,
           perc = perc - x) %>% 
    select(cons, perc) %>% 
    as.matrix()
  
  K <- diag(dnorm(Z[,2] / h))
  
  beta <- solve(t(Z) %*% K %*% Z) %*% t(Z) %*% K %*% reg_data[-val]$totalscore
  return(t(beta))
}

get_loo_error <- function(val, h){
  
  x <- reg_data[val]$perc
  y <- reg_data[val]$totalscore
  
  beta_vals <- get_ll_beta(val, x, h)
  
  error <- y - beta_vals[, 'cons']
  
  return(error)
}

coarse_grid_results <- purrr::map_dfr(h_grid_coarse, function(h_val){
  errors <- purrr::map_dbl(1:nrow(reg_data), function(val){
    return(get_loo_error(val, h_val))
  })
  return(c('h_val' = h_val, 'loss' = mean(errors^2)))
})

min_h_coarse <- coarse_grid_results %>% 
  filter(loss == min(loss)) %>% 
  pull(h_val)

h_grid_fine <- seq(min_h_coarse - 1, min_h_coarse + 1, by = .2)

fine_grid_results <- purrr::map_dfr(h_grid_fine, function(h_val){
  errors <- purrr::map_dbl(1:nrow(reg_data), function(val){
    return(get_loo_error(val, h_val))
  })
  return(c('h_val' = h_val, 'loss' = mean(errors^2)))
})

h_cv <- fine_grid_results %>% 
  filter(loss == min(loss)) %>% 
  pull(h_val)

cat(str_interp('We get a CV bandwidth of `h_cv = ${h_cv}`.'))
```

We get a CV bandwidth of `h_cv = 9.4`.

### Calculate Clustered Cross-Validation Bandwidth

This will be fairly similar to the previous calculation, except that we
need to cluster by school. The nice thing is that instead of leaving out
an observation, we leave out a cluster, so there should be fewer
computations.

``` r
h_grid_coarse <- seq(4, 20, by = 1)

reg_data_clustered <- reg_data %>%
  group_by(schoolid) %>% 
  nest() %>% 
  ungroup()

get_ll_beta_nc <- function(school_var, x, h){

  data_rel <- reg_data_clustered %>%
    filter(schoolid != school_var) %>%
    mutate(mats = map(data, function(df){
            Z <- df %>%
              mutate(cons = 1,
                     perc = perc - x) %>%
              select(cons, perc) %>%
              as.matrix()

            Y <- df %>%
              pull(totalscore)

            K <- diag(dnorm(Z[,'perc'] / h))

            return(list('Z' = Z, 'Y' = Y, 'K' = K))
          }),
          inv_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$Z)
          }),
          num_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$Y)
          }))

  beta <- solve(Reduce('+', data_rel$inv_term)) %*% Reduce('+', data_rel$num_term)

  return(t(beta))
}

get_nc_error <- function(val, h){
  
  x <- reg_data[val]$perc
  y <- reg_data[val]$totalscore
  school <- reg_data[val]$schoolid
  
  beta_vals <- get_ll_beta_nc(school, x, h)
  
  error <- y - beta_vals[, 'cons']
  
  return(error)
}

coarse_grid_results <- purrr::map_dfr(h_grid_coarse, function(h_val){
  start <- Sys.time()
  errors <- purrr::map_dbl(1:nrow(reg_data), function(val){
    return(get_nc_error(val, h_val))
  })
  return(c('h_val' = h_val, 'loss' = mean(errors^2)))
})

min_h_coarse <- coarse_grid_results %>% 
  filter(loss == min(loss)) %>% 
  pull(h_val)

h_grid_fine <- seq(min_h_coarse - 1, min_h_coarse + 1, by = .2)

fine_grid_results <- purrr::map_dfr(h_grid_fine, function(h_val){
  errors <- purrr::map_dbl(1:nrow(reg_data), function(val){
    return(get_nc_error(val, h_val))
  })
  return(c('h_val' = h_val, 'loss' = mean(errors^2)))
})

h_cvc <- fine_grid_results %>% 
  filter(loss == min(loss)) %>% 
  pull(h_val)

cat(str_interp('We get a CV clustered bandwidth of `h_cvc = ${h_cvc}`.'))
```

We get a CV clustered bandwidth of `h_cvc = 9`.

### Calculation of Local Linear Estimator With Different Kernels

We use the same local linear estimation procedure as above, but using
the kernels that we have calculated for clustered and non-clustered
bootstrap.

``` r
get_ll_beta_cluster <- function(x, h){

  data_rel <- reg_data_clustered %>%
    mutate(mats = map(data, function(df){
            Z <- df %>%
              mutate(cons = 1,
                     perc = perc - x) %>%
              select(cons, perc) %>%
              as.matrix()

            Y <- df %>%
              pull(totalscore)

            K <- diag(dnorm(Z[,'perc'] / h))

            return(list('Z' = Z, 'Y' = Y, 'K' = K))
          }),
          inv_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$Z)
          }),
          num_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$Y)
          }))

  beta <- solve(Reduce('+', data_rel$inv_term)) %*% Reduce('+', data_rel$num_term)

  return(as.numeric(beta[1]))
}

plot_data <- data.table(x = c(0:100)) %>% 
  rowwise() %>% 
  mutate(ll_cv = get_ll_beta_cluster(x, h_cv),
         ll_cvc = get_ll_beta_cluster(x, h_cvc)) %>% 
  ungroup() %>% 
  pivot_longer(cols = ll_cv:ll_cvc, names_to = 'bw', values_to = 'y')

ggplot(plot_data) +
  geom_line(aes(x = x, y = y, linetype = bw)) +
  scale_x_continuous(limits = c(0,100), 
                     breaks = seq(0,100,10),
                     expand = c(0,0)) + 
  scale_y_continuous(limits = c(5,27.5), 
                     breaks = seq(5,25,5),
                     expand = c(0,0)) +
  scale_linetype_manual(labels = c('LL Using Conventional CV Bandwidth',
                                   'LL Using Clustered CV Bandwidth'),
                        values = c('dashed',
                                   'solid')) +
  labs(x = 'Percentile',
       y = 'Testscore') + 
  theme_bw() + 
  theme(legend.position = 'bottom', legend.title = element_blank())
```

![](hansen_19_07_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

### Add Standard Errors

As before, we compute our delete-cluster errors and then use them to
compute the variance-covariance matrix.

``` r
reg_data_w_errors <- reg_data %>% 
  mutate(e = map_dbl(c(1:nrow(reg_data)), function(val){
    return(get_nc_error(val, h_cvc))
    })) %>% 
  group_by(schoolid) %>% 
  nest() %>% 
  ungroup()

get_vcov <- function(x, h){

  data_rel <- reg_data_w_errors %>%
    mutate(mats = map(data, function(df){
            Z <- df %>%
              mutate(cons = 1,
                     perc = perc - x) %>%
              select(cons, perc) %>%
              as.matrix()

            Y <- df %>%
              pull(totalscore)

            K <- diag(dnorm(Z[,'perc'] / h))
            
            e <- df %>% 
              pull(e)

            return(list('Z' = Z, 'Y' = Y, 'K' = K, 'e' = e))
          }),
          inv_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$Z)
          }),
          num_term = map(mats, function(val){
            return(t(val$Z) %*% val$K %*% val$e %*% t(val$e) %*% val$K %*% val$Z)
          }))

  v_cov <- solve(Reduce('+', data_rel$inv_term)) %*% Reduce('+', data_rel$num_term) %*% solve(Reduce('+', data_rel$inv_term))

  return(v_cov)
}

get_se <- function(x, h){
  return(sqrt(get_vcov(x,h)[1,1]))
}

plot_data_ses <- data.table(x = c(0:100)) %>% 
  rowwise() %>% 
  mutate(testscore = get_ll_beta_cluster(x, h_cvc),
         se = get_se(x, h_cvc)) %>% 
  ungroup()

ggplot(data = plot_data_ses) + 
  geom_line(aes(x = x, y = testscore)) + 
  geom_smooth(aes(x = x, y = testscore),
              method = 'lm', 
              formula=y~x, 
              linetype='dashed',
              se = FALSE,
              color = 'black',
              size = .5)+
  geom_ribbon(aes(x = x, 
                  ymin = testscore - 1.96*se, 
                  ymax = testscore + 1.96*se), 
              alpha = .3, 
              fill = 'dodgerblue3') +
  scale_x_continuous(limits = c(0,100), 
                     breaks = seq(0,100,10),
                     expand = c(0,0)) + 
  scale_y_continuous(limits = c(0,30), 
                     breaks = seq(0,30,5),
                     expand = c(0,0)) +
  labs(x = 'Percentile',
       y = 'Testscore') + 
  theme_bw()
```

![](hansen_19_07_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->
