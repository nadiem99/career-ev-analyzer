# Career EV Analyzer: Methodology

This document describes the data sources, modeling approach, and key assumptions behind the Career EV (Expected Value) Analyzer. Every parameter is grounded in publicly available compensation data and historical market returns.

> **Currently implemented in the dashboard:** Consulting, Big Tech (Generalist), High-Growth Startup, Corp Strategy / Finance, Found a Company. Investment Banking, Private Equity, and Venture Capital are documented below for reference and may be re-introduced later.

---

## 1. Model Overview

The Career EV Analyzer computes the **expected monetary value** of a career path over a configurable time horizon (typically 10 years). It accounts for:

- **Annual cash compensation** (base salary + bonus + signing bonuses amortized)
- **Annual equity grants** valued at an expected long-run multiplier
- **Retention probability** (the likelihood the person is still on that path in a given year)
- **Exit compensation** (what someone earns if they leave the path for a fallback role)
- **Founder liquidity events** (discrete payouts modeling lumpy startup outcomes)

All dollars are discounted to present value at **5% per year**, applied uniformly across on-path cash, on-path equity, and exit fallback comp — so the comparison between staying and leaving is apples-to-apples.

### Core formula

For each year `y` in `[0, horizon-1]`:

```
year_value(y) =
      stay(y) * [ cash(y) + grant(y) * mult(y) * (1 - P_zero(y)) ]
    + (stay(y-1) - stay(y)) * NPV_exit(y)
    + stay(y) * founder_event(y)

total_EV = sum over y of year_value(y) / (1.05)^y
```

Where:

| Symbol | Meaning |
|---|---|
| `stay(y)` | Probability the person is still on this path at year `y` (`stay(-1) = 1`) |
| `cash(y)` | Cash compensation (salary + bonus) in year `y` |
| `grant(y)` | Face value of equity granted in year `y` |
| `mult(y)` | Expected long-run multiplier on a grant given in year `y` |
| `P_zero(y)` | Probability the equity goes to zero (failure, severe drawdown) |
| `NPV_exit(y)` | Present value (at year `y`) of earning the year-y exit comp every year through the horizon |
| `founder_event(y)` | Year-y founder liquidity event: `value(y) * prob(y)` |

### Exit compensation: NPV model

When someone leaves a path in year `y`, their outside option is valued as the **net present value at year y** of following the exit-compensation trajectory through the remaining horizon, discounted at 5% per year:

```
NPV_exit(y) = sum over k in [y..horizon-1] of:
    exit_comp(k) / (1 + 0.05)^(k - y)
```

This NPV is then itself discounted back to year 0 by the outer `/ (1.05)^y` factor, so the overall comparison between paths is in today's dollars.

The `exit_comp` array is therefore a **year-by-year fallback trajectory**, not a single flat number: leave at Y3 and you start earning the year-3 exit comp; if you stay in fallback through Y10 you earn the year-10 exit comp. This reflects that ex-consultants, ex-Big-Tech etc. continue to advance in their fallback careers (corp dev associate → director → VP, for example).

### Role adjustments

Cash and equity values are multiplied by **role-specific adjustment factors** to model differences between roles at the same company tier (e.g., a PM vs. a Chief of Staff at a FAANG company). Adjustments are static — promotions and role transitions are not modeled.

### Carry / liquidity event model (founders)

The founder path does not use annual equity grants. Instead, it models **discrete payout events** with associated values and probabilities per year:

```
founder_event(y) = stay(y) * value(y) * prob(y)
```

Each year is treated as an independent shot at a liquidity event (seed raise, Series A, growth round, exit). The per-year EVs sum to the total expected founder-equity value over the horizon. This is a back-of-envelope simplification — a real founder outcome is one big lumpy event, not a string of small ones — but the per-year decomposition gives a reasonable expected total. The same mechanism would apply to PE/VC carry if those paths are re-introduced (PE carry typically starts ~Y5; VC carry ~Y7 due to 10-year fund cycles).

---

## 2. Compensation Data Sources

All compensation figures are benchmarked against the following public sources:

### Tech Compensation

| Source | Coverage | Used For |
|---|---|---|
| Levels.fyi 2025 End of Year Pay Report | Industry-wide tech compensation benchmarks | Calibrating overall TC ranges across tiers |
| Levels.fyi company-specific pages | Google PM, Meta PM, Nvidia, OpenAI, Anthropic, Anduril, Databricks, Stripe | Individual company cash and equity splits |
| Carta State of Startup Compensation H1 2025 | Startup salary and equity grant benchmarks | Series A-B and Series C-D cash/equity calibration |
| TopStartups.io Startup Salary & Equity Database | Startup compensation across stages | Cross-referencing startup equity grant sizes |

### Consulting / Non-Tech Compensation

| Source | Coverage | Used For |
|---|---|---|
| Management Consulted 2026 Salary Report | MBB and tier-2 consulting firms | Post-MBA associate and engagement manager comp |
| CaseLane McKinsey Salary 2025 | McKinsey-specific salary bands | McKinsey cash comp by level |
| Poets&Quants Consulting Pay 2025 | Cross-firm consulting compensation | Benchmarking non-McKinsey consulting paths |
| Oliver Wyman on Levels.fyi | Oliver Wyman salary data | Tier-2 consulting calibration |

### Finance Compensation (IB, PE, VC)

| Source | Coverage | Used For |
|---|---|---|
| Wall Street Oasis Compensation Reports | IB analyst/associate/VP/MD comp | Investment banking cash + bonus benchmarks |
| Heidrick & Struggles PE/VC Compensation Survey | PE and VC comp across fund sizes | PE base + bonus, VC base + bonus calibration |
| Preqin / PitchBook Fund Performance Data | PE/VC fund return distributions | Calibrating carry payout probabilities and timing |
| Levels.fyi Finance | Corporate finance, FP&A, corp dev comp | Corp strategy/finance path calibration |

### Corporate Strategy / Finance

| Source | Coverage | Used For |
|---|---|---|
| Levels.fyi (F500 companies) | Strategy, FP&A, corp dev roles at large public companies | Cash comp and RSU grant benchmarks |
| Glassdoor / Blind salary data | Cross-company corporate role compensation | Supplementary comp verification |

---

## 3. Equity Multiplier Methodology

Equity multipliers represent how much a dollar of granted equity is worth at each future year. A multiplier of `1.0` means the equity retains its grant-date value; `2.0` means it doubles; `0.0` means total loss.

Each company tier has three scenario trajectories calibrated against historical precedent:

### 3.1 FAANG / Mega-Cap Tech

| Scenario | Year 1 | Year 10 | Rationale |
|---|---|---|---|
| Bear | 0.9x | 0.55x | Meta fell 76.7% from peak (Sep 2021) to trough (Nov 2022). A prolonged bear case reflects a major drawdown with only partial recovery over the decade. |
| Base | 1.0x | 3.0x | Implies ~12% CAGR, consistent with FAANG portfolio 10-year annualized returns of ~27% (which includes the extraordinary run-up; 12% is a conservative forward estimate for established mega-caps). |
| Bull | 1.1x | 5.0x | A strong performer growing from an already large base, comparable to Apple or Nvidia's growth from established positions (not from small-cap). |

### 3.2 High-Growth Public Tech (e.g., Nvidia)

| Scenario | Year 1 | Year 10 | Rationale |
|---|---|---|---|
| Bear | 0.7x | 0.15x | Cisco rose 3,800% from 1995-2000, then fell 88% (from $79 to $9.50) and took 25 years to reclaim its peak. A similar correction from today's elevated valuations would be devastating. Amazon fell 94% during the dot-com bust. |
| Base | 1.0x | 5.0x | Implies ~18% CAGR. Nvidia's 5-year return was 1,282% (~13.8x), but compounding from a $3T+ market cap is harder. 18% CAGR is an optimistic-but-plausible base. |
| Bull | 1.2x | 11.0x | Implies ~27% CAGR, matching historical FAANG portfolio returns. Capped because sustaining hyper-growth from a $3T starting point is historically unprecedented. For reference, Nvidia's 10-year return was 22,492% (~226x), but that started from a much smaller base. |

### 3.3 Late-Stage Private (Series C-D, e.g., Stripe, Databricks)

| Scenario | Year 1 | Year 10 | Rationale |
|---|---|---|---|
| Bear | 0.8x | 0.10x | Down rounds and declining value. Stripe peaked at ~$95B (2021), dropped to ~$50B (2023) -- a ~47% haircut. With liquidation preferences, common shareholders can be wiped out even without total company failure. Participating preferences and 2x clauses rose from 29.6% (2022) to 47.0% (2023). |
| Base | 1.0x | 3.0x | A solid IPO at 1.5-3x the last private round. Databricks rose from $43B (2023) to $62B (late 2024), a ~1.4x increase in ~1 year. Stripe recovered to ~$91.5B via a Feb 2025 tender offer and reached ~$129B on secondary markets by Dec 2025, implying ~1.8-2.6x from the 2023 trough over two years. Carta data shows the median step-up multiple at Series C was only 1.6x in Q2 2024. |
| Bull | 1.0x | 5.5x | Category winner that dominates its market and achieves a premium IPO or acquisition. |

### 3.4 Early-Stage Startup (Series A-B)

| Scenario | Year 1 | Year 10 | Rationale |
|---|---|---|---|
| Bear | 0x | 0x | Total failure. CB Insights data shows ~35% of Series A companies shut down entirely, 67% of VC-backed companies stall, and 75% never return cash to investors. The bear case reflects zero recovery. This outcome has ~60% probability weight in the model. |
| Base | 1.0x | 20.0x | Moderate exit after dilution. A successful Series A company that reaches exit returns meaningful value, but dilution is severe: Index Ventures data shows employees owning 1% at seed are diluted to ~0.40% by Series D (60% dilution). The 20x multiplier on the grant-date value accounts for the company's growth net of this dilution. |
| Bull | 1.0x | 50.0x | Breakout company -- a category-defining startup that achieves a large exit. Capped at 50x because even extraordinary outcomes are bounded by the ~60% dilution employees experience through successive funding rounds (Index Ventures). |

### 3.5 Additional Context on Equity Liquidity

- Only 32% of vested in-the-money options were exercised in late 2024 (Carta), reflecting liquidity constraints and tax friction that reduce the realized value of startup equity.
- Nvidia experienced a -50.3% drawdown in 2022 despite its long-term trajectory, illustrating that even the best-performing stocks have severe interim volatility.

---

## 4. Survival / Retention Rates

The `stayProb` curve models the probability that someone remains on a given career path at each year. It declines over time, reflecting voluntary attrition, layoffs, burnout, and career pivots.

### 4.1 Management Consulting (MBB)

MBB firms operate an "up or out" system with ~18-25% annual attrition. Median post-MBA tenure is approximately 2.5 years. Only ~1-2% of a post-MBA cohort eventually makes Partner.

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.85 | ~15% leave Y1 (voluntary + early managed exits) |
| 2 | 0.65 | First major "up or out" cycle; median tenure ~2.5yr means <50% remain by mid-Y3 |
| 3 | 0.48 | Second exit wave post-EM promotion decisions |
| 4 | 0.35 | Only partner-track candidates remain |
| 5 | 0.25 | AP-level thinning |
| 10 | 0.06 | Approaching partner; <5-10% of original cohort |

**Sources:** CaseCoach (up-or-out policy analysis), Strat-Bridge (McKinsey partner promotion data), industry tenure surveys.

### 4.2 Investment Banking

IB has extremely high attrition — most analysts and associates leave within 2-3 years for PE, hedge funds, or corporate roles. Only a small fraction reach MD level (~8% at Y10).

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.82 | ~18% leave Y1 (some pre-planned 2yr stints) |
| 2 | 0.55 | Massive exit wave — most 2yr analyst-to-PE moves happen here |
| 3 | 0.35 | Associate level; second wave of departures |
| 5 | 0.20 | Only VP-track lifers remain |
| 10 | 0.08 | MD level; ~8% of original cohort |

### 4.3 Private Equity

PE has high junior attrition (2-year associate programs) but lower senior attrition (partners stay for carry).

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.90 | Associates generally stay through program |
| 2 | 0.75 | End of associate program; many return to bschool or switch |
| 5 | 0.42 | VP level; carry starts vesting |
| 10 | 0.21 | Principal/Partner track; carry retention effect |

### 4.4 Venture Capital

VC is a small industry with moderate attrition. Associate/principal roles turn over, but partners tend to stay.

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.90 | Low initial attrition |
| 2 | 0.78 | Some associates rotate out |
| 5 | 0.50 | Principal level decisions |
| 10 | 0.28 | Partners with carry stake |

### 4.5 Corp Strategy / Finance

F500 corporate roles have the highest retention — stable comp, predictable hours, low involuntary attrition.

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.93 | Very low initial attrition |
| 5 | 0.72 | Steady; some move to startups or MBA |
| 10 | 0.50 | ~50% still in corp path after a decade |

### 4.6 Startup Founders

Founder retention combines two failure modes: (a) the company fails entirely, and (b) the founder is replaced even if the company survives.

- **Company survival:** BLS data shows ~52% of businesses survive to year 5 and ~35% to year 10. VC-backed startups fail at higher rates — CB Insights reports 67% stall and 75% never return cash to investors (Harvard Business School).
- **Founder replacement:** 20-40% of founders are replaced by investors over the company's life. Among unicorns, ~35% of founders are no longer CEO (Harvard Law School/HBR research).

The combined `stayProb` is approximately `P(company alive) × P(founder still CEO | company alive)`. For a VC-backed founder persona, we use roughly 25–35% company survival at Y10 and ~65–70% founder retention given survival, yielding ~17–20% at Y10:

| Year | stayProb | Notes |
|------|----------|-------|
| 1 | 0.78 | ~20% early failures + a few founder departures |
| 2 | 0.62 | ~35% cumulative failure; founder replacement begins |
| 3 | 0.50 | ~40% companies gone; some replacement |
| 4 | 0.42 | ~45% gone; replacement accelerates |
| 5 | 0.35 | ~50% companies dead × ~70% founder retention |
| 6 | 0.30 | continuing decline |
| 7 | 0.26 | continuing decline |
| 8 | 0.22 | continuing decline |
| 9 | 0.19 | continuing decline |
| 10 | 0.17 | ~25–30% company survival × ~65% founder retention |

**Sources:** BLS Establishment Age and Survival Data, CB Insights Venture Capital Funnel, Harvard Business School (VC failure rates), HBR/Harvard Law (founder replacement research).

### 4.7 Tech and Startups (Employee)

| Path | Y1 | Y5 | Y10 | Basis |
|------|-----|-----|------|-------|
| FAANG | 0.93 | 0.70 | 0.44 | Median tenure 2-4yr; lower attrition than startups |
| High-Growth Tech | 0.92 | 0.65 | 0.38 | Slightly higher attrition than FAANG |
| Series A-B | 0.85 | 0.48 | 0.21 | Higher company failure + startup attrition |
| Series C-D | 0.90 | 0.60 | 0.33 | More stable than early-stage; pre-IPO retention incentives |

### 4.8 Scenario Probability Normalization

In detailed mode, bear/base/bull scenario probabilities are **auto-normalized to sum to 100%** at each year. When a user edits one scenario's probability, the other two are proportionally scaled so the total remains 1.0. This ensures the EV calculation is always mathematically valid.

The exit-compensation mechanism ensures that attrition is not purely value-destroying: people who leave a path continue earning in their fallback role, valued as an NPV of the outside option (see Section 1).

---

## 5. Key Assumptions and Limitations

### Assumptions

| Assumption | Justification |
|---|---|
| **Equity grants are annual and vest linearly** | Standard 4-year vesting with annual refresh grants at major tech companies. Startup grants may vest on different schedules, but annual is a reasonable approximation. |
| **Exit compensation grows over time** | People who leave one path (e.g., a startup) typically take roles with increasing pay as they gain seniority. The model reflects this with a growing exit comp array. |
| **Scenarios are independent of retention** | The model does not correlate equity outcomes with attrition (e.g., a bear market causing more people to leave). In reality, these are correlated. |
| **No taxes are modeled** | All values are pre-tax. Effective tax rates vary significantly by jurisdiction, equity type (ISO vs. NSO vs. RSU), and holding period. |
| **Uniform 5% discount rate** | Applied to all dollars (on-path cash, on-path equity, and exit fallback comp). Roughly the long-run real return on a balanced portfolio — a reasonable opportunity cost. Users should pick a different rate if they have one in mind. |
| **Vesting is not modeled** | A grant in year `y` is counted at its expected value in year `y`, not over its actual vesting schedule. For tech RSUs (4-year vest), this slightly overweights early-year EV; for startups with cliff-and-vest, the simplification is rougher. |
| **Equity multiplier interpretation** | `mult(y)` represents the long-run expected value of $1 of equity granted in year `y`, by the time it's realized. The model does not separately track each grant's appreciation across subsequent years. |
| **Exit comp follows the array trajectory after departure** | If you leave at year 3, the model walks through `exit_comp[3..9]` for the remaining years — i.e., your fallback comp grows. The array values reflect typical career growth in the fallback role (e.g., corp dev associate → director → VP). |
| **Role adjustments are static** | The model applies fixed role-adjustment multipliers rather than modeling promotions or role changes dynamically. |

### Limitations

1. **Compensation data has survivorship and self-reporting bias.** Levels.fyi data skews toward higher-paying companies and toward employees willing to share. Actual medians may be lower.
2. **Equity outcomes are fat-tailed.** The three-scenario model (bear/base/bull) is a simplification. Real equity returns have extreme outliers in both directions that discrete scenarios cannot fully capture.
3. **Correlation is ignored.** In practice, a market downturn affects equity values, retention, and exit compensation simultaneously. The model treats these as independent.
4. **Startup outcomes are bimodal.** The Series A-B model uses a high bear-case probability (~60%) to reflect this, but the smooth blending of scenarios still understates how binary early-stage outcomes tend to be.
5. **Liquidity constraints are not modeled.** Startup equity may be illiquid for years. The model values equity at its theoretical multiplied value without accounting for the inability to sell, exercise costs, or tax events (AMT on ISOs, etc.). As noted, only 32% of vested in-the-money options were exercised in late 2024.
6. **Dilution is embedded in multipliers, not modeled explicitly.** For startups, the equity multiplier trajectories already account for expected dilution (per Index Ventures data), but they do not model round-by-round dilution dynamically.
7. **No geographic or cost-of-living adjustment.** Compensation figures reflect top-market (Bay Area / NYC) levels and are not adjusted for other regions.

---

## Sources

1. Levels.fyi. *2025 End of Year Pay Report.* https://www.levels.fyi/2025/
2. Levels.fyi. Company-specific compensation data (Google, Meta, Nvidia, OpenAI, Anthropic, Anduril, Databricks, Stripe). https://www.levels.fyi/
3. Management Consulted. *2026 Consulting Salary Report.* https://managementconsulted.com/
4. CaseLane. *McKinsey Salary 2025.* https://caselane.com/
5. Poets&Quants. *Consulting Pay 2025.* https://poetsandquants.com/
6. Carta. *State of Startup Compensation H1 2025.* https://carta.com/
7. Carta. *State of Private Markets Q2 2024* (step-up multiples, option exercise rates).
8. TopStartups.io. *Startup Salary & Equity Database.* https://topstartups.io/
9. CB Insights. *Venture Capital Funnel* (Series A failure rates, VC-backed company outcomes).
10. Index Ventures. *Employee equity dilution benchmarks* (seed-to-Series D dilution).
11. Historical market data: Nvidia, Cisco, Meta, Amazon stock price histories (Yahoo Finance, Google Finance).
12. Liquidation preference trends: various VC industry reports (2022-2023 data on participating preferences).
