# Report: NY Income Tax Data Exploration (1999–2013)

*Results below come from an actual run of `income_tax_analysis.py` against
the synthetic dataset described in `README.md`. Numbers will shift slightly
if you regenerate the data with a different random seed, and will change
more substantially if you swap in the real dataset — but the qualitative
conclusions (which model fits better, which counties are outliers, etc.)
are written to illustrate the kind of story you should expect to tell with
your own results.*

## (a) Exploring Tompkins County

**Plot 1 — Number of returns by income bracket, 1999–2013**
`figures/a1_returns_by_bracket.png`

A line plot, one line per bracket. Each bracket is a continuous time series
sampled once a year, and the interesting feature is the *trend* in each
bracket as well as how brackets compare to each other — lines make both easy
to read at once. A bar chart (10 bars × 15 years) would be too busy, and a
histogram doesn't apply here at all — a histogram shows the distribution of
one variable, not a value tracked over time.

**Plot 2 — Average tax per return, disregarding income class**
`figures/a2_avg_tax_over_time.png`

Also a line plot: there is one (year, avg_tax) point per year, and the thing
we care about is the trend across consecutive years. A line connects
consecutive years so the eye follows the trend (including the dip around
2008–2009); a scatter plot would show the same points without that visual
continuity, and a bar chart tends to visually overstate small year-to-year
differences relative to a line.

The series itself rises from about **$1,046** (1999) to **$1,831** (2013),
dips noticeably in **2008–2009** (recession), and then rises sharply through
2010–2012.

## (b) Linear model: year → avg tax

Feature map φ(x) = [year, 1].

```
w_b = [55.00, -108,898.67]
avg_tax(year) ≈ 55.00 · year − 108,898.67
```

Training MSE ≈ **6,294** (RMSE ≈ **$79**). A single straight line captures
the broad upward trend but, by construction, cannot bend for the
2008–2009 dip or the post-2009 acceleration.

## (c) Add avg tax from the previous year

Feature map φ(x) = [year, avg_tax₍t-1₎, 1].

```
w_c = [28.09, 0.494, -55,604.01]
avg_tax_t ≈ 28.09 · year + 0.494 · avg_tax_(t-1) − 55,604.01
```

Training MSE ≈ **5,103** (RMSE ≈ **$71**) — better than part (b).

**Interpretation:**
- The **0.494** coefficient on avg_tax₍t-1₎ says roughly half of this year's
  average tax "persists" from last year's value, on top of whatever the
  year trend explains — i.e., tax levels are noticeably *sticky* year to
  year, but not perfectly (a coefficient of 1 would mean "this year = last
  year, shifted").
- The year coefficient (28.09, smaller than part (b)'s 55.00) captures
  whatever *additional* linear drift isn't already absorbed by the lag term.
- The intercept has no standalone meaning here (year = 0 isn't meaningful).

**Fit plot:** `figures/c_pred_vs_actual.png`

The model tracks the overall shape well but lags the real turning points by
about one year — it doesn't predict the 2008 drop until 2009, and doesn't
predict the 2010–2011 rebound until it's already underway. This is exactly
what you'd expect from a model whose only memory of the recent past is a
single one-year-old number: it can react to a shock, but only with a
one-year delay.

## (d) Two new features (only using data through year t−1)

**Feature space:**

```
X_d = [ year, avg_tax_(t-1), total_returns_(t-1), top_bracket_share_(t-1), 1 ]
```

- **`total_returns_(t-1)`** — total county returns filed *last* year, a
  proxy for local economic size/growth. Fully known by the end of year t−1.
- **`top_bracket_share_(t-1)`** — the fraction of *last* year's returns in
  the "$200,000 or more" bracket, a proxy for how income-heavy the filer mix
  was. Income composition tends to persist year to year, and high earners
  contribute disproportionately to tax revenue, so this should help predict
  this year's average beyond the lag and year terms alone.

Both are computed strictly from data through year t−1, satisfying the
"prediction in year t depends only on data through t−1" requirement.

```
w_d = [21.20, 0.677, -0.0304, 26689.21, -41723.16]
        year    lag1    returns_lag1   top_bracket_lag1   intercept
```

Training MSE ≈ **3,675** (RMSE ≈ **$61**) — better than both (b) and (c).

**Interpretation:**
- `total_returns_(t-1)` coefficient (−0.030) is tiny in raw terms, but that's
  expected: this feature is on the scale of tens of thousands while avg_tax
  is on the scale of thousands of dollars. Its *sign* (slightly negative)
  suggests that, holding the other features fixed, a county that had more
  filers last year saw slightly *lower* average tax this year in this
  data — consistent with "more returns" sometimes meaning more lower/middle
  earners filing, not necessarily a richer filer base.
- `top_bracket_share_(t-1)` (≈ 26,689) is interpretable more directly,
  since the feature itself is a proportion (roughly 0–0.05 in this data): a
  one-percentage-point increase in last year's top-bracket share predicts
  about a **$267** increase in this year's average tax — a large, intuitive
  effect, since the top bracket pays disproportionately more tax.

## (e) Comparing wc and wd

| Model | avg_tax₍t-1₎ coefficient |
|---|---|
| (c) | 0.494 |
| (d) | 0.677 |

The coefficient **increases** by about 0.18 when the two new features are
added. That's the opposite of the naive guess that new features would
"steal" explanatory credit from the lag term. The likely explanation: part
of the year-to-year noise that diluted the apparent persistence in model (c)
is actually correlated with county size and income composition. Once
`total_returns_(t-1)` and `top_bracket_share_(t-1)` are allowed to absorb
that part of the variation, the *partial* relationship between last year's
tax level and this year's comes through more cleanly — i.e., the "true"
persistence effect was being masked, not inflated, by omitting those two
features in model (c).

(If you re-run this against the real dataset and instead see the
coefficient *shrink*, the story flips: that would mean avg_tax₍t-1₎ in
model (c) was partly standing in for what the new features explain more
directly.)

## (f) Applying wc (Tompkins' model) to other counties

After dropping the 3 fake aggregate rows and the 2 counties with incomplete
1999–2013 coverage (`Hamilton`, `Yates`), **24 counties** remain with full
coverage, including Tompkins.

| | MSE |
|---|---|
| Tompkins (its own model c, in-sample) | 5,103 |
| Other counties — mean | 229,127 |
| Other counties — median | 22,498 |
| Other counties — min / max | 5,297 / 3,178,807 |

**Histogram:** `figures/f_error_histogram_model_c.png` (linear scale — a few
counties' errors are so much larger they squash everything else visually)
and `figures/f_error_histogram_model_c_logscale.png` (log scale — shows the
shape of the bulk of counties).

**Outliers:** the three worst-fit counties are **Manhattan** (MSE ≈
3,178,807), **Westchester** (≈ 714,960), and **Nassau** (≈ 657,718) — the
highest-income counties in the dataset, by a wide margin. This makes sense:
Tompkins' coefficients (especially the intercept and lag coefficient) were
tuned to Tompkins' dollar scale, and applying them to counties whose average
tax is several times higher produces systematically large errors. Most
other counties land much closer to Tompkins' own error, just somewhat
higher — consistent with the model's *trend/persistence* relationship
transferring reasonably, even though its absolute scale doesn't.

## (g) County-specific models (same features as part d)

| | MSE |
|---|---|
| Tompkins (its own model d, in-sample) | 3,675 |
| County-specific models — mean | 5,401 |
| County-specific models — median | 3,700 |

**Histogram:** `figures/g_error_histogram_county_specific.png`

Compared to part (f), the spread of errors is dramatically tighter (low
thousands here, vs. up to several million in part (f)) — letting each
county fit its *own* scale and trend, rather than reusing Tompkins',
removes almost all of the scale-driven error. Tompkins' own MSE sits right
in the middle of this distribution; it is no longer an unusually good fit
relative to the rest, because every county now gets to fit itself.

**Are the coefficients about the same across counties?** No — they vary
quite a bit:

| Coefficient | Std. dev. across counties |
|---|---|
| year | 20.78 |
| avg_tax₍t-1₎ | 0.231 |
| returns₍t-1₎ | 0.078 |
| top_bracket_share₍t-1₎ | 16,660 |
| intercept | 39,876 |

For comparison, Tompkins' own lag coefficient (0.677) is fairly typical, but
counties range from **Bronx** (0.246 — much less persistence) and
**Rockland** (0.249) up to **Madison** (1.135) and **Nassau** (1.072) —
more than a four-fold spread. This says the strength of year-to-year "tax
stickiness" genuinely differs by county, not just the income level.

## (h) County-specific vs. Tompkins model for future predictions

**County-specific models:**
+ Tuned to each county's own scale/trend → best in-sample fit (part g).
− Fit on only ~13–14 yearly points with 5 free parameters — real overfitting
  risk; little "spare" data to confirm a pattern is genuine rather than
  noise specific to that county's history.
− No information about structural changes (e.g. a major local employer
  leaving) beyond what already happened in 1999–2013.

**Tompkins model applied elsewhere (part f):**
+ Its relationship was estimated from one real, representative time series,
  so the *trend/persistence pattern* it captures is at least grounded in
  actual data, not just curve-fit to a handful of points it's being asked to
  predict for.
− Performs badly in absolute terms on counties far from Tompkins' income
  scale (Manhattan, Westchester, Nassau) since the coefficients implicitly
  bake in Tompkins' dollar range.

**Takeaway:** for a county similar to Tompkins in size and wealth, the
Tompkins model's relationship may generalize reasonably without overfitting
that county's short history. For a county very different from Tompkins, a
county-specific model is probably necessary just to get the right *scale*,
but should ideally be fit with more years of data (or pooled with other
counties via county-level features, e.g. average income level or county
size) to reduce overfitting risk rather than fit on each county in total
isolation.

## (i) Other information that would help

- **Local unemployment / employment by sector** — a leading indicator of
  income shocks, available well before the next tax year's filings.
- **Tax law changes** (bracket thresholds, credits) — these shift average
  tax mechanically, independent of income trends, and are known in advance.
- **Cost-of-living / inflation indices** — to separate real income growth
  from nominal, inflation-driven tax growth.
- **Population and migration data** — to distinguish "more returns because
  more people moved in" from "more returns per existing resident."
- **Tompkins-specific:** university enrollment/employment data (Cornell /
  Ithaca College), since a college-town economy can behave differently from
  a typical county's.
- **Same-year statewide or neighboring-county average tax** — regional
  shocks (like the 2008–2009 recession) hit many counties at once; a
  feature like "statewide average tax change last year" could let the model
  react to broad shocks faster than a single county's own one-year lag can.
