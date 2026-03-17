# Recipe and Ratings Analysis

---

## Introduction

This project uses the Recipes and Ratings dataset from food.com, which contains
recipes and user reviews submitted since 2008. The dataset was originally scraped
for a recommender systems paper and includes structured information such as cooking
time, nutrition, and user ratings.

**Central Question:** What is the relationship between cooking time and the average
rating of a recipe?

This question is relevant to anyone contributing recipes to food platforms, as it
may reveal whether time investment in a recipe influences how it is received.

**Number of raw rows:** 83,782

**Relevant Columns:**

| Column | Description |
|--------|-------------|
| `minutes` | Minutes to prepare the recipe |
| `average_rating` | Mean user rating for the recipe (1-5) |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients in the recipe |
| `nutrition` | Nutrition info including calories, fat, sugar, sodium, protein, saturated fat, and carbohydrates |
| `tags` | Food.com tags associated with the recipe |
| `contributor_id` | User ID of the recipe contributor |
| `submitted` | Date the recipe was submitted |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The following cleaning steps were applied to prepare the dataset for analysis:

1. **Left merge of recipes and interactions:** The recipes and interactions datasets
   were leftmerged on `recipe_id`. This preserves all recipes, including those with
   no user reviews, while attaching any available rating data.

2. **Replacing 0 ratings with NaN:** Ratings of 0 were replaced with `np.nan`. A
   rating of 0 is not a valid value on the 1–5 scale and most likely represents a
   missing or unsubmitted rating rather than a genuine score. Keeping these as 0
   would artificially deflate average ratings.

3. **Computing average rating per recipe:** The mean of valid ratings was computed
   per recipe and merged back into the recipes dataset as `average_rating`. This
   produces one row per recipe with a summary rating for modeling and analysis.

4. **Removing extreme outliers (minutes > 1000):** Recipes with cooking times above
   1000 minutes (~16.7 hours) were removed as likely data entry errors. A threshold
   of 1000 was chosen rather than a lower value because some legitimate recipes —
   such as BBQ brisket, smoked meats, and slow brines — can genuinely require 12–16
   hours. Values extending into the hundreds of thousands or millions of minutes are
   not realistic and substantially distort the distribution.

After cleaning, the dataset contains **80,552 rows**.

**Head of cleaned DataFrame:**

| | name | id | minutes | contributor_id | description | ingredients | n_ingredients | average_rating |
|---|---|---|---|---|---|---|---|---|
| 0 | 1 brownies in the world best ever | 333281 | 40 | 985201 | these are the most; chocolatey, moist, rich, d... | ['bittersweet chocolate', 'unsalted butter', '... | 9 | 4.0 |
| 1 | 1 in canada chocolate chip cookies | 453467 | 45 | 1848091 | this is the recipe that we use at my school ca... | ['white sugar', 'brown sugar', 'salt', 'margar... | 11 | 5.0 |
| 2 | 412 broccoli casserole | 306168 | 40 | 50969 | since there are already 411 recipes for brocco... | ['frozen broccoli cuts', 'cream of chicken sou... | 9 | 5.0 |
| 3 | millionaire pound cake | 286009 | 120 | 461724 | why a millionaire pound cake? because it's su... | ['butter', 'sugar', 'eggs', 'all-purpose flour... | 7 | 5.0 |
| 4 | 2000 meatloaf | 475785 | 90 | 2202916 | ... | ... | ... | ... |

### Univariate Analysis

<img width="595" height="393" alt="image" src="https://github.com/user-attachments/assets/b39887ae-2cd2-456b-b2aa-39ad33ea06e4" />


Average ratings are concentrated toward the higher end of the scale, with most
recipes receiving ratings between 4 and 5 stars. This strong positive skew suggests
users tend to rate recipes favorably, which compresses the range of the response
variable and makes prediction harder.

### Bivariate Analysis

<img width="583" height="394" alt="image" src="https://github.com/user-attachments/assets/ca601457-9025-402b-9809-4a98ef8fde6c" />

The plot does not show a strong linear relationship between cooking time and average
rating. Ratings appear relatively stable across cooking times, consistent with the
hypothesis test result that cooking time has little practical effect on user ratings.

### Interesting Aggregates

| time_group | mean | median | std | count |
|---|---|---|---|---|
| Longer than Median | 4.61 | 5.0 | 0.66 | 38980 |
| Shorter than or Equal to Median | 4.64 | 5.0 | 0.62 | 41572 |

Recipes with shorter cooking times have a slightly higher mean rating (4.64) than
longer ones (4.61). However, both groups share the same median of 5.0, suggesting
cooking time alone may not be a strong driver of user ratings.

---

## Assessment of Missingness

### NMAR Analysis

The `rating` column is likely NMAR (Not Missing at Random). Users who had a negative
or mediocre experience may be less likely to leave a rating, meaning the missingness
is tied to the unobserved value itself. No observed column in the dataset can fully
account for this pattern. Additional data such as user engagement logs would be needed
to explain it and make it MAR.

The merged dataset has three columns with missing values: `rating`, `description`, and
`average_rating`. Since `average_rating` is derived from `rating`, its missingness is a
direct consequence of the same process. `description` was chosen for the permutation
tests below as it has nontrivial missingness independent of the rating process.

### Missingness Dependency

Permutation tests (1000 iterations, two sided) were run to assess whether the
missingness of `description` depends on other observed columns.

<img width="594" height="400" alt="image" src="https://github.com/user-attachments/assets/9bb03384-e92c-4cfb-a03c-9ac66668ecb3" />

- `n_ingredients` (p = 0.0000): The null is rejected at α = 0.05. Recipes with
  missing descriptions tend to have fewer ingredients on average (obs. diff = -1.13).
  Missingness is dependent on this column.
- `n_steps` (p = 0.2300): The null is not rejected at α = 0.05. Missingness shows
  no significant relationship with number of steps (obs. diff = 0.72).

This supports that `description` is MAR: its missingness can be explained by other
observed variables rather than the missing value itself.

---

## Hypothesis Testing

**Null Hypothesis (H0):** There is no difference in average rating between recipes
with cooking times above the median and recipes at or below the median.

**Alternative Hypothesis (H1):** Recipes with cooking times above the median have
a higher average rating than shorter recipes.

**Test Statistic:** Difference in mean average rating (longer minus shorter recipes).
The mean is appropriate here because average_rating is a continuous numeric outcome
and the question is about whether one group systematically rates higher than another.
The difference in means directly measures that gap.

**Why a one sided test:** The alternative hypothesis is directional as it predicts
that longer recipes rate higher, not just that the two groups differ. A onesided
test is the right choice when there is a specific predicted direction rather than
just a general claim of difference.

**Why a permutation test:** No assumptions are made about the underlying distribution
of ratings. Given that ratings are heavily skewed toward 5 stars, a parametric test
like a t-test would be less appropriate. A permutation test is distribution free and
applicable regardless of the shape of the data.

<img width="597" height="396" alt="image" src="https://github.com/user-attachments/assets/b1b5956b-c219-4cf1-a6af-330388fc7a98" />

**P-value (one sided):** 1.0000

**Conclusion:**
The observed difference in mean rating (long minus short) was -0.0321, meaning shorter
recipes had a slightly higher average rating. At a significance level of α = 0.05, with
a one sided p-value of 1.0000, there is no evidence to reject the null hypothesis. The
data does not support the claim that longer recipes tend to receive higher ratings than
shorter ones. If anything, the direction of the observed difference is opposite to what
the alternative hypothesis predicted, though the magnitude is too small to draw any
strong practical conclusions.

---

## Framing a Prediction Problem

**Prediction Task:** Predict the average rating of a recipe (regression).

**Response Variable:** `average_rating` — the mean user rating for a recipe, ranging
from 1 to 5. This was chosen because understanding what makes a recipe highly rated
is useful for recipe contributors and recommendation systems.

**Evaluation Metric:** RMSE (Root Mean Squared Error). RMSE was chosen because it
penalizes larger prediction errors more heavily than MAE. A prediction off by 2 stars
is much worse than one off by 0.5 stars, and RMSE captures that better.

**Features and time of prediction justification:** All features are known when a
recipe is first posted, before any user has rated it:

| Feature | Type | Description |
|---------|------|-------------|
| `minutes` | Quantitative | Cooking time, submitted with the recipe |
| `n_steps` | Quantitative | Number of steps, part of the recipe itself |
| `n_ingredients` | Quantitative | Number of ingredients, part of the recipe itself |
| `calories` | Quantitative | Derived from the nutrition field, submitted with the recipe |

Interaction-based features such as number of reviews are excluded since those would
not exist at the time of prediction.

---

## Baseline Model

The baseline model is a Linear Regression using four quantitative features: `minutes`,
`n_steps`, `n_ingredients`, and `calories`. There are no ordinal or nominal features
in this baseline, so no categorical encoding was needed. Features were standardized
using StandardScaler before fitting, all implemented in a single sklearn Pipeline.

**Feature breakdown:**
- Quantitative: 4 (`minutes`, `n_steps`, `n_ingredients`, `calories`)
- Ordinal: 0
- Nominal: 0

**Performance:**
- Train RMSE: 0.6398
- Test RMSE: 0.6401
- Dummy (mean) RMSE: 0.6400

The baseline model is not good. Its test RMSE is essentially identical to the dummy
regressor that simply predicts the mean rating for every recipe. This means the four
numeric recipe features carry almost no predictive signal for average rating on their
own. The close train and test RMSE values indicate the model generalizes well, but
only because it has learned very little. A more expressive model with richer features
is needed in the final model step.

---

## Final Model

### Features Added

**Nutrition (7 features):** All seven values from the nutrition field are parsed
individually: calories, total fat, sugar, sodium, protein, saturated fat, and
carbohydrates. These are quantitative and used as is. Nutritional content is a
property of the recipe itself and likely influences how users perceive and rate it.
A high calorie dessert and a low calorie salad attract different audiences with
different rating tendencies.

**Ratio features (7 features):** Derived from nutrition and recipe structure.
For example, 

protein_ratio = protein / (calories + 1),

fat_protein_ratio,
cal_per_minute, recipe_simplicity, and time_efficiency. Raw values alone do not
capture how nutrients relate to each other or to the recipe's structure. A recipe
with 500 calories means something different if it takes 10 minutes versus 2 hours.
These ratios capture those relationships in a way a linear model on raw values
would miss.

**Text features via TF-IDF + SVD:** Recipe name, description, tags, ingredients,
and steps are each vectorized with TF-IDF and compressed to dense SVD components
(10-20 per field). The text a contributor writes reflects the recipe's identity and
intended audience. A recipe titled "quick weeknight pasta" signals something
different from "traditional Neapolitan ragu," and those signals may correlate with
rating patterns. SVD keeps the feature count manageable while preserving the
dominant semantic structure.

**Target encoded categoricals (nominal):** The top 50 tags, top 30 ingredients,
and contributor_id are target encoded using sklearn's TargetEncoder. Tags and
ingredients define what category a recipe belongs to, and certain categories
systematically attract higher or lower ratings regardless of the specific recipe.
contributor_id captures the fact that some contributors have a track record of
posting recipes that rate well, which is a real signal in the data generating
process. One hot encoding would be impractical given the high cardinality.

**Date features (2):** submit_year and submit_month capture temporal patterns in
how recipes are rated. User behavior and rating norms on food platforms shift over
time, and recipes submitted in certain periods may reflect different community
standards or platform demographics.

**Other numeric features:** name_length, desc_length, avg_step_length,
avg_ingredient_rarity, n_tags, log_minutes. The effort a contributor puts into
writing a recipe, measured by description length and step detail, may signal
quality or care that users respond to. Ingredient rarity captures whether a recipe
uses niche or hard to find items, which could influence ratings from users who
struggled to source them.

All features are available at the time a recipe is posted. No interaction based
features are used.

### Model and Hyperparameter Tuning

HistGradientBoostingRegressor was chosen over Linear Regression because it handles
nonlinear relationships, missing values natively, and scales well to higher
dimensional feature sets.

The following hyperparameters were tuned using Optuna with 100 trials and 5 fold
cross validation, optimizing for RMSE:

- **max_depth:** controls tree depth. Deeper trees capture more complex patterns but
  risk overfitting.
- **learning_rate:** controls the step size at each iteration. Lower values generalize
  better but require more iterations.
- **max_iter:** number of boosting rounds.
- **min_samples_leaf:** minimum samples at a leaf node. Higher values reduce overfitting.
- **l2_regularization:** directly penalizes model complexity.
- **max_bins:** number of bins for histogram construction.

**Best hyperparameters found:**

| Hyperparameter | Value |
|---|---|
| max_depth | 7 |
| learning_rate | 0.04106 |
| max_iter | 450 |
| min_samples_leaf | 72 |
| l2_regularization | 0.4827 |
| max_bins | 128 |

Best CV RMSE: 0.6300

### Performance

| Model | Test RMSE |
|---|---|
| Dummy (mean) | 0.6400 |
| Baseline (Linear Regression, 4 features) | 0.6401 |
| Final Model (HistGBR, all recipe features) | 0.6312 |

<img width="599" height="400" alt="image" src="https://github.com/user-attachments/assets/3e5b2da3-1c2c-4b2d-80f1-ecbcd620c461" />


The final model achieves a test RMSE of 0.6302, compared to 0.6401 for both the
baseline and the dummy predictor. The improvement is modest but consistent with
what the data allows. Recipe level features at post time carry limited information
about future user ratings, largely because the rating distribution is heavily skewed
toward 5 stars.

The top features by permutation importance are the target encoded contributor_id
and tag/ingredient embeddings, suggesting that who posts the recipe and what
category it falls into matter more than its specific nutritional or structural
properties.

---

## Fairness Analysis

**Groups:**
- Group X: "Quick recipes" — cooking time at or below the median (35 minutes)
- Group Y: "Slow recipes" — cooking time above the median (35 minutes)

**Evaluation Metric:** RMSE

**Null Hypothesis:** The model is fair. Its RMSE for quick recipes and slow recipes
are roughly the same, and any observed difference is due to random chance.

**Alternative Hypothesis:** The model is unfair. Its RMSE for slow recipes is higher
than its RMSE for quick recipes.

**Test Statistic:** RMSE(slow) - RMSE(quick)

**Significance Level:** α = 0.05

<img width="591" height="394" alt="image" src="https://github.com/user-attachments/assets/00ef3aea-a194-4553-8410-502734902c5b" />

**Results:**
- RMSE (slow recipes): 0.6580
- RMSE (quick recipes): 0.6032
- Observed difference: 0.0548
- P-value (one sided): 0.0000

**Conclusion:**
The observed RMSE for slow recipes (0.6580) was higher than for quick recipes (0.6032),
a difference of 0.0548. At a significance level of α = 0.05, with a one sided p-value
of 0.0000, there is sufficient evidence to reject the null hypothesis. The data suggests
the model performs worse on slow recipes than on quick ones.

This is consistent with the overall findings of the project. Slow recipes likely include
more varied and niche categories such as brines, fermented foods, and slow cooked meats,
which may have less predictable rating patterns than the more common quick recipes that
dominate the dataset. The model appears to struggle more in that space.
