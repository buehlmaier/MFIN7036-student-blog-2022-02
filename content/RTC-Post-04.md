---
Title: Part 4: Problems and solutions in Analysis part (Group Raw Text Connoisseurs)
Date: 2022-03-29 14:50
Category: Analysis Report

---

By Group "Raw Text Connoisseurs"

## Problem 1

Our purpose is to explore whether there is a relationship between keywords and corporate profitability. The model determined at the beginning is the Hausman test (fixed effect model). The model design is as follows:
$$
ROE = F(keywordscore, controls) = C + \beta_1 \cdot keywordscore + \beta_2 \cdot LNGDP + \beta_3 \cdot AVEROE
$$

1. LNGDP (Gross National Product Logarithm): measure the level of economic development in the current year.

2. AVEROE (Industry average return on equity): measure the average level of return on equity in the corresponding industry.

Using STATA for regression, get error: "More samples needed". That is, the sample size is too small to use the fixed effects model. The error still occurs after expanding the sample from the top 10 companies to 20 companies.

## solution

The linear regression model is used instead. Although it is not as accurate as the Hausman test, it is convenient to operate and is also suitable for smaller samples.



## Problem 2

After performing linear regression, it is found that the significance of the model is generally strong, which is not in line with expectations. The preliminary judgment is that there is a problem with the keyword classification, and the keyword extraction is not accurate enough.

## solution

Manually filter keywords to eliminate irrelevant words and reduce their interference. After filtering, the rationality of the model results is significantly improved. And it shows in the following highlighted words as an example.

['climate', 'energy', 'infrastructure', 'investing', 'emission', **'http',** 'development', 'carbon', **'paris'**, **price'**, **'project'**, 'technology', 'government', 'tax', 'mitigation', 'fuel', 'iea', 'asset', **'increase'**, ...]