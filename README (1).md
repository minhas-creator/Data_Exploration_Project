# Project 3: Data Exploration — NY Income Tax (1999–2013)

## ⚠️ About the dataset — please read this first

I (Claude) could not reach `data.ny.gov` from my sandboxed environment to
download the real NY State Department of Taxation and Finance file. The real
dataset this assignment is based on appears to be:

**"Income Tax Components by Size of Income by Place of Residence: Beginning
Tax Year 1999"** — NY State Open Data, dataset ID `5bb2-yb85`
https://data.ny.gov/Government-Finance/Income-Tax-Components-by-Size-of-Income-by-Place-o/5bb2-yb85

Since I couldn't fetch it, **`generate_dataset.py` procedurally builds a
synthetic CSV with the same structure**: one row per (county, year, income
bracket) with a returns count, total NY AGI, and total tax liability — plus
the same "gotchas" the real data has:

- Three rows that *look* like counties but are aggregates/catch-alls
  (`"Grand Total, Full-Year Resident"`, `"NYS Unclassified +"`,
  `"Residence Unknown ++"`) that part (f) explicitly tells you to exclude.
- Two counties with **incomplete** 1999–2013 coverage (`Hamilton` only
  reports from 2004 on; `Yates` is missing 2002 entirely), so the
  "only counties with full 1999-2013 data" filter in part (f) actually does
  something.

The dollar figures are **not real NY tax numbers** — they come from a
believable generative model (income-bracket progressivity, county wealth
differences, a 2008–2009 recession dip, year-over-year persistence) chosen
so that every part of the assignment produces sensible, interpretable
results. **If you get the real `incomeTax.ipynb` / CSV from Google
Classroom, swap it in** — `income_tax_analysis.py` only depends on the
column names below, so it should run unchanged (see "Using the real
dataset" at the bottom).

## Files

| File | Purpose |
|---|---|
| `generate_dataset.py` | Builds `ny_income_tax_1999_2013.csv` (synthetic data) |
| `ny_income_tax_1999_2013.csv` | The dataset itself |
| `income_tax_analysis.py` | The full analysis — parts (a) through (i) |
| `REPORT.md` | Write-up of results/answers, generated from an actual run |
| `figures/` | All plots, saved as PNGs (created when you run the analysis script) |

## Dataset columns

| Column | Meaning |
|---|---|
| `Tax Year` | 1999–2013 |
| `County` | NY county name (or one of the 3 fake aggregate labels — see above) |
| `Income Class` | One of 10 income brackets (`"Under $5,000"` … `"$200,000 or more"`), or `"All Classes"` for the fake aggregate rows |
| `Number of Returns` | Count of tax returns filed in that county/year/bracket |
| `NY AGI Total ($)` | Total New York Adjusted Gross Income for that county/year/bracket, in dollars |
| `NYS Tax Liability Total ($)` | Total NY State tax liability for that county/year/bracket, in dollars |

**Average tax per return** for a county-year (the quantity the whole project
is about) is *not* a column — you compute it as:

```
avg_tax = (sum of NYS Tax Liability Total over all brackets)
        / (sum of Number of Returns over all brackets)
```

This is exactly what `county_year_table()` in `income_tax_analysis.py` does.

## How to run

```bash
pip install numpy pandas matplotlib
python3 generate_dataset.py        # creates ny_income_tax_1999_2013.csv
python3 income_tax_analysis.py     # runs parts (a)-(i), saves figures/
```

In VS Code: open `income_tax_analysis.py`, install the **Jupyter** extension
if you don't have it, and run cell-by-cell using the `# %%` markers (each one
is a separate notebook cell — click "Run Cell" above each `# %%` line, or use
the Python Interactive Window). This is also exactly how it'll convert into
`incomeTax.ipynb` — each `# %%` block becomes one notebook cell — so once
you're happy with it, send the file back and ask for the conversion.

## Using the real dataset

If you get the actual file from Google Classroom, you mainly need the column
names in `income_tax_analysis.py` to line up:

1. Set `DATA_PATH` at the top of `income_tax_analysis.py` to your real file.
2. Make sure your file has (or rename to) these column names: `Tax Year`,
   `County`, `Income Class`, `Number of Returns`, and a tax-liability column
   — if yours is named differently (e.g. `NYS Tax Liability` or `Tax
   Liability ($000)`), update the two spots in `county_year_table()` and
   `build_d_features()` that reference `"NYS Tax Liability Total ($)"`.
3. Update `FAKE_COUNTY_NAMES` to match however your real file labels its
   "Grand Total" / "Unclassified" / "Unknown" rows (check `df["County"]
   .unique()` to see the exact strings).
4. Everything else (the modeling code, the plots, the discussion structure)
   needs no changes — only the column-name plumbing is dataset-specific.
