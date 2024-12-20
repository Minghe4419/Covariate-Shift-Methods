diabetes_example
================
Minghe Wang
2024-11-21

# Diabete Dataset

``` r
# Sample sizes
n_source <- 500
n_target <- 500

# True coefficients for diabetes risk
beta_0 <- -10
beta_1 <- 0.05  # Age coefficient
beta_2 <- 0.2   # BMI coefficient
beta_3 <- 0.03  # BP coefficient

# Generate Source Data (Age 30-50)
age_source <- runif(n_source, min = 30, max = 50)
bmi_source <- rnorm(n_source, mean = 25, sd = 3)
bp_source <- rnorm(n_source, mean = 120, sd = 10)

# Compute probability of diabetes
logit_p_source <- beta_0 + beta_1 * age_source + beta_2 * bmi_source + beta_3 * bp_source
p_diabetes_source <- 1 / (1 + exp(-logit_p_source))

# Assign diabetes status
diabetes_source <- rbinom(n_source, size = 1, prob = p_diabetes_source)

# Create source data frame
data_source <- data.frame(
  Age = age_source,
  BMI = bmi_source,
  BP = bp_source,
  Diabetes = diabetes_source
)

# Generate Target Data (Age 50-70)
age_target <- runif(n_target, min = 50, max = 70)
bmi_target <- rnorm(n_target, mean = 28, sd = 4)
bp_target <- rnorm(n_target, mean = 130, sd = 12)

# Create target data frame (Diabetes status unknown)
data_target <- data.frame(
  Age = age_target,
  BMI = bmi_target,
  BP = bp_target
)

# Visualize distributions
library(reshape2)
data_source_melt <- melt(data_source[, c('Age', 'BMI', 'BP')])
```

    ## No id variables; using all as measure variables

``` r
data_target_melt <- melt(data_target[, c('Age', 'BMI', 'BP')])
```

    ## No id variables; using all as measure variables

``` r
data_source_melt$Dataset <- 'Source'
data_target_melt$Dataset <- 'Target'

data_combined <- rbind(data_source_melt, data_target_melt)

ggplot(data_combined, aes(x = value, fill = Dataset)) +
  geom_density(alpha = 0.5) +
  facet_wrap(~variable, scales = 'free') +
  labs(title = 'Feature Distributions in Source and Target Datasets')
```

![](diabetes_data_testing_files/figure-gfm/data_generating-1.png)<!-- -->

Here we generate source and target datasets containing 3 input variables
`Age`, `BMI`, `BP` and binary output variable `Diabetes`(only in source
data).

# Separate density estimation

``` r
# Estimate densities using kernel density estimation
# Estimating density for each feature in the source and target datasets
# Using 'ks' package for kernel density estimation

# Kernel density estimation for Age
f_hat_source_age <- kde(x = data_source$Age)
f_hat_target_age <- kde(x = data_target$Age)

# Kernel density estimation for BMI
f_hat_source_bmi <- kde(x = data_source$BMI)
f_hat_target_bmi <- kde(x = data_target$BMI)

# Kernel density estimation for BP
f_hat_source_bp <- kde(x = data_source$BP)
f_hat_target_bp <- kde(x = data_target$BP)

# Estimate density ratios for each feature
# Extracting estimated densities at each point in source dataset
source_density_age <- predict(f_hat_source_age, x = data_source$Age)
target_density_age <- predict(f_hat_target_age, x = data_source$Age)
w_hat_age <- target_density_age / source_density_age

source_density_bmi <- predict(f_hat_source_bmi, x = data_source$BMI)
target_density_bmi <- predict(f_hat_target_bmi, x = data_source$BMI)
w_hat_bmi <- target_density_bmi / source_density_bmi

source_density_bp <- predict(f_hat_source_bp, x = data_source$BP)
target_density_bp <- predict(f_hat_target_bp, x = data_source$BP)
w_hat_bp <- target_density_bp / source_density_bp

# Calculate combined weights
combined_weights <- w_hat_age * w_hat_bmi * w_hat_bp

# Print some values of the combined weights
head(combined_weights)
```

    ## [1] 4.568274e-05 5.628761e-05 1.789256e-05 1.076829e-02 2.234706e-01
    ## [6] 2.122685e-05

``` r
# Visualize density ratios for Age, BMI, and BP
ggplot(data.frame(Age = data_source$Age, Density_Ratio = w_hat_age), aes(x = Age, y = Density_Ratio)) +
  geom_line() +
  labs(title = 'Estimated Density Ratios for Age', x = 'Age', y = 'Density Ratio')
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

``` r
ggplot(data.frame(BMI = data_source$BMI, Density_Ratio = w_hat_bmi), aes(x = BMI, y = Density_Ratio)) +
  geom_line() +
  labs(title = 'Estimated Density Ratios for BMI', x = 'BMI', y = 'Density Ratio')
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-2.png)<!-- -->

``` r
ggplot(data.frame(BP = data_source$BP, Density_Ratio = w_hat_bp), aes(x = BP, y = Density_Ratio)) +
  geom_line() +
  labs(title = 'Estimated Density Ratios for BP', x = 'BP', y = 'Density Ratio')
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-3.png)<!-- -->

``` r
# Visualize true densities for comparison
# Plot the true densities of Age, BMI, and BP for both source and target populations
# Plot the KDE estimates alongside the true distributions for comparison
ggplot() +
  geom_line(data = data.frame(Age = f_hat_source_age$eval.points, Density = f_hat_source_age$estimate), aes(x = Age, y = Density), color = 'blue', linetype = "dashed", size = 1, alpha = 0.7) +
  geom_density(data = data_source, aes(x = Age, y = ..density..), color = 'blue', fill = 'blue', alpha = 0.3) +
  geom_line(data = data.frame(Age = f_hat_target_age$eval.points, Density = f_hat_target_age$estimate), aes(x = Age, y = Density), color = 'red', linetype = "solid", size = 1, alpha = 0.7) +
  geom_density(data = data_target, aes(x = Age, y = ..density..), color = 'red', fill = 'red', alpha = 0.3) +
  labs(title = 'True vs KDE Estimated Densities of Age for Source and Target Populations', x = 'Age', y = 'Density') +
  scale_color_manual(name = "Dataset", values = c("Source" = "blue", "Target" = "red")) +
  theme_minimal()
```

    ## Warning: Using `size` aesthetic for lines was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `linewidth` instead.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

    ## Warning: The dot-dot notation (`..density..`) was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `after_stat(density)` instead.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

    ## Warning: No shared levels found between `names(values)` of the manual scale and the
    ## data's colour values.

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-4.png)<!-- -->

``` r
# Visualize reweighted source distribution compared to target
data_source_weighted <- data_source

data_source_weighted$weights <- combined_weights

ggplot() +
  geom_density(data = data_source_weighted, aes(x = Age, weight = weights), color = 'blue', fill = 'blue', alpha = 0.3) +
  geom_density(data = data_target, aes(x = Age), color = 'red', fill = 'red', alpha = 0.3) +
  labs(title = 'Reweighted Source vs Target Distribution for Age', x = 'Age', y = 'Density') +
  theme_minimal()
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-5.png)<!-- -->

``` r
ggplot() +
  geom_density(data = data_source_weighted, aes(x = BMI, weight = weights), color = 'blue', fill = 'blue', alpha = 0.3) +
  geom_density(data = data_target, aes(x = BMI), color = 'red', fill = 'red', alpha = 0.3) +
  labs(title = 'Reweighted Source vs Target Distribution for BMI', x = 'BMI', y = 'Density') +
  theme_minimal()
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-6.png)<!-- -->

``` r
ggplot() +
  geom_density(data = data_source_weighted, aes(x = BP, weight = weights), color = 'blue', fill = 'blue', alpha = 0.3) +
  geom_density(data = data_target, aes(x = BP), color = 'red', fill = 'red', alpha = 0.3) +
  labs(title = 'Reweighted Source vs Target Distribution for BP', x = 'BP', y = 'Density') +
  theme_minimal()
```

![](diabetes_data_testing_files/figure-gfm/unnamed-chunk-1-7.png)<!-- -->
