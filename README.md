# Career EV Analyzer

An interactive tool for comparing the expected value of different career paths — consulting, big tech, startups, and founding a company.

## What it does

- Models career paths with year-by-year compensation, equity grants, survival probabilities, and exit role comp
- **Simple view**: single equity multiplier across 4 paths (Consulting, Big Tech, Startup, Founder)
- **Detailed view**: bear/base/bull equity scenarios across 7 paths (MBB, Oliver Wyman, FAANG, High-Growth Tech, Series A-B Startup, Series C-D Startup, Founder)
- Role selector: PM, Chief of Staff, Strategy & Ops, BizOps — adjusts cash and equity by role
- Every assumption is editable
- URL updates in real time — share your exact config by copying the link

## How to use

1. Visit the site
2. Toggle between Simple and Detailed views
3. Select your role type and time horizon
4. Click any career path card to drill down into year-by-year breakdown
5. Expand any path in the Assumptions section to tweak numbers
6. Click "Share this config" to copy your personalized link

## Deploy your own

This is a single HTML file with no build step. To deploy:

1. Fork this repo
2. Go to Settings → Pages → Source: Deploy from branch → `main` / `root`
3. Your site will be live at `https://yourusername.github.io/career-ev-analyzer/`

## Methodology

**Exit Role Comp**: When you leave a track (e.g., don't make Partner), the model pays exit comp for all remaining years in the horizon.

**Simple model**: `EV = P(on track) × [cash + grant × mult × P(eq≠0)] + P(off track) × exit_comp × remaining_years`

**Detailed model**: Replaces single multiplier with bear/base/bull scenarios. `EV = P(on track) × [cash + grant × Σ(scenario_mult × scenario_prob)] + P(off track) × exit_comp × remaining_years`

**What's not captured**: Time value of money, taxes, cost of living, vesting schedules, non-financial value (learning, network, optionality).
