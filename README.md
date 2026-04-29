# Career EV Analyzer

An interactive tool for comparing the expected value of different career paths — consulting, big tech, startups, and founding a company.

## What it does

- Models five career paths year-by-year: **Consulting**, **Big Tech (Generalist)**, **High-Growth Startup**, **Corp Strategy / Finance**, **Found a Company**
- Combines cash compensation, equity grants (with an expected long-run multiplier), retention probability, and the fallback value of an exit role into a single expected-value number per path
- Discounts every dollar — on-path cash, on-path equity, and exit fallback comp — uniformly at 5%/year, so comparisons are in today's dollars
- Role selector: PM, Chief of Staff, Strategy & Ops, BizOps — adjusts cash and equity by role
- Every assumption is editable, and your config is reflected in the URL — copy the link to share your exact setup

## How to use

1. Visit the site
2. Pick your role and time horizon (5 / 7 / 10 years)
3. Toggle which paths to compare
4. Click any card to drill into year-by-year cash vs equity EV
5. Expand any path in "Assumptions by path" to tweak numbers
6. Expand the "Methodology & Assumptions" panel at the bottom for the full formula and what's not captured

## Deploy your own

This is a single HTML file with no build step. To deploy:

1. Fork this repo
2. Go to Settings → Pages → Source: Deploy from branch → `main` / `root`
3. Your site will be live at `https://yourusername.github.io/career-ev-analyzer/`

## Methodology in brief

For each year on a path:

```
year_value = stay × [cash + grant × mult × (1 - P_zero)]
           + (prev_stay - stay) × NPV_exit
           + stay × founder_event
total_EV   = Σ year_value(y) / (1.05)^y
```

`NPV_exit(y)` is the present value at year `y` of following the exit-comp trajectory `exit_comp[y..horizon-1]` — your fallback comp grows over the remaining years, just like a real career would.

**Not captured:** taxes, cost of living, vesting cliffs, AMT on ISOs, illiquidity of private equity, correlation between market downturns and attrition, non-financial value (network, learning, optionality). It's a back-of-envelope, not a forecast.

See [METHODOLOGY.md](METHODOLOGY.md) for full data sources and per-path calibration notes.
