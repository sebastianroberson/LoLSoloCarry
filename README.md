# League of Legends Solo Carry Analysis

**Author**: Sebastian Roberson

---

League of Legends Solo Carry Analysis is a data science project conducted at UCSD. The project includes the stages starting from exploratory data analysis to hypothesis testing, baseline models, and fairness analysis. The focus of this project is to investigate the roles that are most likely to "solo carry" in League of Legends professional matches and its impact on gold and kill share participation. 

## Introduction

### Dataset

This project uses the [Oracle's Elixir](https://oracleselixir.com/) League of Legends
esports match dataset, spanning three seasons (2024 – 2026). The reason for only choosing three seasons is to limit probability of the results being due to a previous "meta" of champions which could affect the results. Each row describes one
player's performance in a single professional match. There are also "team" summary rows
(one per team per game) that we exclude from player-level analysis.

### Research Question

> **Which role (position) in professional League of Legends is most likely to be the
> "solo carry" — the single player whose individual performance most drives their
> team's victory?**

We define the **solo carry** as the player on the *winning* team who dealt the highest
fraction of their team's total damage to champions (`damageshare`) in that game.
This definition captures the player whose offensive output stood out most among teammates.

### Why It Matters

The "can X role carry alone?" debate is central to how players prioritise champion
selection, in-game decision-making, and macro strategy. A data-driven answer
across thousands of elite matches reveals genuine structural tendencies rather than
anecdote.

### Relevant Columns

| Column | Type | Description |
|--------|------|-------------|
| `position` | nominal | Player role: top / jng / mid / bot / sup (or "team" for aggregate rows) |
| `result` | binary | Game outcome: 1 = win, 0 = loss |
| `damageshare` | quantitative | Fraction of team damage dealt to champions by this player |
| `earnedgoldshare` | quantitative | Fraction of team earned gold (excludes starting gold) by this player |
| `kills` | quantitative | Champion kills scored by this player |
| `deaths` | quantitative | Number of times this player died |
| `assists` | quantitative | Kill assists contributed by this player |
| `teamkills` | quantitative | Total champion kills scored by this player's team |
| `cspm` | quantitative | Minion + monster kills per minute (farming efficiency) |
| `vspm` | quantitative | Vision score per minute (warding / vision control) |
| `gamelength` | quantitative | Game duration in seconds |
| `league` | nominal | Regional tournament / league (e.g., LCS, LCK, LPL, LEC) |
| `datacompleteness` | nominal | "complete" = all stats present; "partial" = early-game stats missing |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw data contains two types of rows:
1. **Player rows** (`position` ∈ {top, jng, mid, bot, sup}) — one row per player per game.
2. **Team rows** (`position == 'team'`) — aggregate summaries per team per game.

We also distinguish `datacompleteness`:
- `'complete'` — all statistics are populated (majority of rows).
- `'partial'` — rows from leagues that don't report early-game timestamps;
  many columns (e.g., `goldat10`) are `NaN`.

**Cleaning steps:**

| Step | Action | Rationale |
|------|--------|-----------|
| 1 | Drop `position == 'team'` rows | We analyse individual players, not teams |
| 2 | Keep `datacompleteness == 'complete'` for the main analysis | Ensures all feature columns are populated |
| 3 | Retain all player rows (incl. partial) in `players_all` | Used for missingness analysis in Step 3 |
| 4 | Engineer `kda`, `kill_participation`, `gamelength_min`, `dmg_gold_ratio` | Derived performance metrics needed for modelling |
| 5 | Label `is_carry` | Binary carry label: 1 for the winning player with max `damageshare` per game |

No column was imputed or dropped other than the structural filtering above.

### Univariate Analysis

**Plot 1** — Carry rate per position reveals a dramatic skew: bot and mid lane account for
the vast majority of solo carries, while support almost never holds that title.

<iframe
  src="assets/carry-rate-by-position.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Plot 2** — Distribution of `damageshare` by position confirms that bot and mid
systematically occupy higher damage-share values than utility roles.

<iframe
  src="assets/damage-share-by-position.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Bivariate Analysis

`damageshare` and `earnedgoldshare` are both strongly associated with role, and
together they create well-separated clusters by position — the support cluster
sits in the lower-left (low damage, low gold), while bot/mid occupy the upper-right.

<iframe
  src="assets/damage-vs-gold-share.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Interesting Aggregates

The pivot table below shows, for each position, the mean carry rate alongside
several key performance metrics. Bot lane leads in carry rate, damage share,
and kill participation, while support has the highest vision score per minute (vspm).

| position | Carry Rate | Avg Damage Share | Avg Gold Share | Avg KDA | Avg Kill Participation | Avg CSPM | Avg VSPM | N Players |
|----------|------------|-----------------|----------------|---------|----------------------|----------|----------|-----------|
| bot | 0.218 | 0.269 | 0.260 | 4.258 | 0.657 | 9.039 | 1.074 | 45,562 |
| mid | 0.166 | 0.260 | 0.233 | 3.917 | 0.636 | 8.593 | 1.048 | 45,562 |
| top | 0.089 | 0.225 | 0.211 | 3.002 | 0.513 | 7.848 | 0.965 | 45,562 |
| jng | 0.025 | 0.164 | 0.192 | 4.024 | 0.703 | 6.280 | 1.406 | 45,562 |
| sup | 0.001 | 0.082 | 0.103 | 3.638 | 0.712 | 1.145 | 3.415 | 45,562 |

---

## Assessment of Missingness

### NMAR Analysis

The `url` column stores a broadcast or match-detail link and is missing for
**~89.7 %** of all player rows.

I believe this column is likely **NMAR (Not Missing at Random)**. Whether a match
has a documented URL depends on the match's *prestige and visibility* — specifically,
whether the match was broadcast on a major streaming platform and whether that
platform maintains a stable archive link. "Match prestige" and "streaming-platform
contract" are *not* columns in the dataset; the missingness therefore depends on a
latent factor unobserved in our data.

To convert this to MAR we would need an additional column such as:
- *Peak concurrent viewers* (high viewership → URL recorded), or
- *Broadcast platform* (major platforms like Twitch/YouTube have stable URLs;
  local/regional streams may not).

> Note: Although major leagues (LPL, LDL) show URL coverage = 100%, most leagues
> show 0 % — consistent with prestige-driven URL availability, not pure league
> labelling. The NMAR argument holds because within the "large league" stratum
> individual match prestige (finals vs. regular-season week 1) still affects URL
> availability in ways not captured by `league` alone.

### Missingness Dependency Tests

We test two columns from `players_all` (including partial rows):

| Test | Column tested | Hypothesised | |
|------|--------------|-------------|---|
| 1 | `league` | url missingness **depends on** league | TVD permutation |
| 2 | `result` | url missingness **does not depend on** result | diff-in-means permutation |

**Test 1** — URL missingness vs. `league` (observed TVD = 0.984, p = 0.000 → **Dependent**)

<iframe
  src="assets/missingness-vs-league.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Test 2** — URL missingness vs. `result` (observed diff = 0.00017, p = 0.964 → **Not dependent**)

<iframe
  src="assets/missingness-vs-result.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

---

## Hypothesis Testing

### Question

Does the **bot lane carry significantly more often than the mid lane**, or is the
observed difference explainable by random chance?

### Hypotheses

- **H₀**: Bot and mid lane players carry at the same rate.
  Any observed difference in carry rates is due to random variation.
- **H₁**: Bot lane players carry more often than mid lane players (one-sided).

### Setup

- **Test statistic**: Difference in carry rates — `bot_carry_rate − mid_carry_rate`
- **Method**: Permutation test — shuffle the `position` label among bot/mid players,
  recompute the carry-rate difference for each shuffle.
- **Significance level**: α = 0.05

### Justification

We chose a one-sided test because the prior hypothesis (from game design: bot carries
gold-efficient marksmen while mid plays burst mages) already suggests a direction.
The difference in means is an ideal test statistic because it is directly interpretable
and has no assumptions about the underlying distribution.

**Result** — Bot: 21.8%, Mid: 16.6%, Observed diff: +0.052, p-value: 0.000 → **Reject H₀**

<iframe
  src="assets/hypothesis-bot-vs-mid.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

---

## Framing a Prediction Problem

### Prediction Problem

**Binary classification**: Given a player's end-of-game performance statistics,
predict whether they were the **solo carry** (`is_carry = 1`) of their match.

### Response Variable

`is_carry` — 1 if the player had the highest `damageshare` on the winning team
in their game; 0 otherwise.

**Why this variable?** It directly operationalises our research question and is
computable from observable match data without requiring subjective judgement.

### Evaluation Metric

**F1-score** (on the positive "carry" class).

- The class is imbalanced (~10 % positive), so *accuracy* would be misleading —
  a classifier that always predicts "not carry" achieves 90 % accuracy yet is useless.
- F1 balances **precision** (when we predict carry, are we right?) and **recall**
  (do we catch all carries?), which is the appropriate trade-off here.

### Information Available at "Time of Prediction"

All features used are derived from **end-of-game** statistics; none incorporate
the `result` column, information about other players' scores, or future information.
The prediction simulates a post-game analysis where all per-player stats are known
but we want to identify who carried.

---

## Baseline Model

### Model Description

A **Logistic Regression** classifier wrapped in a single `sklearn` `Pipeline`.

| Feature | Type | Encoding |
|---------|------|----------|
| `damageshare` | Quantitative | Pass-through (already in [0,1]) |
| `earnedgoldshare` | Quantitative | Pass-through (already in [0,1]) |
| `position` | Nominal | One-Hot Encoding (drop first, 4 binary columns) |

`class_weight='balanced'` corrects for the 10:1 class imbalance.

**Why Logistic Regression?** It is a strong, interpretable baseline for binary
classification. The two quantitative features are already normalised fractions,
so additional scaling is unnecessary.

### Is It "Good"?

The baseline achieves moderate F1 (see output below). The two share-based features
are directly related to the carry definition, so a simple linear boundary captures
much of the signal. However, logistic regression cannot model non-linear interactions
(e.g., "high damage share AND high kill participation AND specific position"), leaving
room for improvement.

**Baseline F1 Score (Carry class): 0.5276**

---

## Final Model

### Engineered Features (beyond one-hot encoding)

| Feature | Formula | Motivation |
|---------|---------|-----------|
| `kill_participation` | (kills + assists) / teamkills | Measures combat *involvement*, not just damage — a carry who enables kills also shows up here |
| `kda` | (kills + assists) / (deaths + 1) | Efficiency metric: survives more, kills more → more likely to be the carry |
| `dmg_gold_ratio` | damagetochampions / (earnedgold + 1) | Damage efficiency: generates high damage relative to resources consumed; distinguishes a carry from a "fed tank" |
| `cspm` | (raw from dataset) | Farming output correlates with gold lead, which enables carry performance |
| `vspm` | (raw from dataset) | Vision control distinguishes support from damage roles and helps separate carry from utility |
| `deaths` | (raw from dataset) | Solo carries typically have low death counts; including this directly adds that signal |

### Algorithm: Random Forest Classifier

Random Forests capture **non-linear interactions** between features (e.g., high
`damageshare` AND low `deaths` AND bot lane) that logistic regression misses, making
it well-suited to this problem.

### Hyperparameters Tuned (GridSearchCV, cv=3, scoring='f1')

- `max_depth` — controls tree depth to prevent overfitting
- `min_samples_leaf` — minimum node size; larger values smooth the decision boundary

**Best params**: `max_depth=10`, `min_samples_leaf=5`, `n_estimators=100`

**Final F1 Score (Carry class): 0.6849 — improvement of +0.1573 over baseline**

<iframe
  src="assets/feature-importances.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

---

## Fairness Analysis

### Groups

- **Group X — "Carry roles"**: bot + mid
  (historically high-damage, gold-priority roles)
- **Group Y — "Utility roles"**: top + jng + sup
  (lower expected damage share, different win conditions)

### Metric

**F1-score** on the positive "carry" class — same metric used throughout.

### Hypotheses

- **H₀ (null)**: The model is *fair* — its F1 score is the same for carry roles
  and utility roles. Any observed difference is due to random chance.
- **H₁ (alternative)**: The model is *unfair* — it has a different F1 score for
  carry roles vs. utility roles.

### Method

Permutation test: shuffle the group labels (carry / utility) among test-set players,
recompute |F1(carry) − F1(utility)| for each permutation.

### Significance level

α = 0.05

<iframe
  src="assets/fairness-permutation.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

---
### Conclusion

We reject the H₀ - the model is measurably less accurate at identifying solo carries among utility players than damage-dealing roles. This isn't suprising because carries from bot and mid have high damage share and earned gold share which makes them easier to separate from the non-carries. Utility players who occasionally top the damage charts are rarer and harder to distinguish from the noise, leading to lower F1 for that group. 
