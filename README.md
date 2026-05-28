# CDS Pricing & CS01 Sensitivity

A reduced-form Credit Default Swap pricer with hazard-rate bootstrapping from market spreads, plus a follow-up CS01 sensitivity analysis.

> **Context**: this project was completed as part of a graduate Financial Engineering coursework module on credit risk measurement and management. The base pricer was the deliverable; the CS01 follow-up notebook is an extension done individually.

The base pricer (`cds_pricer.ipynb`) bootstraps a piecewise-constant hazard curve from 10 market CDS spreads via Newton-Raphson with an analytic Jacobian, then prices a non-standard seasoned 30Y CDS with off-cycle quarterly payments. A follow-up notebook (`cs01_analysis.ipynb`) extends this to compute parallel and key-rate CS01 sensitivities across the curve. The full mathematical methodology, derivations, and discussion are in `report.pdf`.

## Headline results

Non-standard CDS, face value $100M, long protection, 28.6Y remaining life as of valuation date 28 Feb 2014:

| | Value (USD) |
|---|---|
| Protection leg | 16,599,748 |
| Premium leg | 24,138,762 |
| **Full MTM** | **−7,539,014** |
| Accrued interest | 51,944 |
| Clean MTM | −7,590,958 |
| **Parallel CS01** | **+115,017** |

Sign convention: MTM is from the **protection buyer's** perspective. Negative MTM indicates the contractual spread (170 bp) exceeds the current fair spread implied by the calibrated curve. Positive CS01 means a parallel widening of CDS spreads benefits the buyer, as expected for long protection.

The CS01 risk concentration is heavily at the long end: the 30Y and 20Y tenors account for ~98% of total parallel CS01. The contract's 28.6Y residual life is the driver — the protection leg integrates default density over the full life, weighted by the discount curve, and there is simply much more contract left at long maturities to be exposed to.

## How to run

```bash
pip install -r requirements.txt
jupyter notebook
```

Run `cds_pricer.ipynb` first (it produces the calibrated hazard curve and the baseline MTM), then `cs01_analysis.ipynb` (which calls `%run cds_pricer.ipynb` to import the calibrated state, then computes the sensitivities).

## Spec assumptions

The base pricer adheres to a fixed set of assignment-imposed conventions:

- **Nelson-Siegel parameters** (β₀, β₁, β₂, τ) = (0.0408, −0.0396, −0.0511, 1.614), supplied directly per spec.
- **Newton-Raphson tolerance** δ = 1×10⁻⁶, initial guess x₀ = 0.01. All bootstrapped residuals fall in the range 1e-7 to 1e-12, well inside tolerance. Convergence is 3-4 iterations per tenor.
- **Subintervals per year** m = 12 for the protection-leg trapezoidal integration. At this density the discretisation error is approximately $10 on $100M notional relative to a daily grid.
- **Recovery rates**: the spec specifies two distinct values for two distinct purposes — **45%** for the calibration basket (the recovery embedded in the market spreads) and **60%** for the non-standard contract (the recovery assumed in pricing). Both are passed through as separate inputs.
- **Day count**: Act/360 for premium accrual fractions; Act/365.25 for indexing the curves at non-standard payment dates. The spec specifies Act/360 explicitly for premium payments but is silent on the year-fraction convention used for curve lookups.
- **Business-day convention**: the spec says "Modified Following" but also specifies no holiday calendar, which collapses the convention to rolling weekends to the next Monday. For the assignment contract no payment date crosses a month boundary, so the strict Modified Following vs Following distinction does not affect results.
- **Survival probability for t > T_M**: the spec defines Q(t) only up to the longest calibration maturity (30Y). For this contract, every Q(t) evaluation falls within the calibrated range (max 28.6Y < 30Y), so the choice of extrapolation policy does not affect reported numbers.

## Modelling assumptions

Beyond the spec-imposed choices, the model class itself involves:

- **Risk-neutral measure**: calibrated hazard rates are risk-neutral and exceed real-world default rates by a risk premium.
- **Independence**: interest rates, default time, and recovery rate are assumed independent. This decoupling enables the closed-form pricing equations.
- **Constant recovery**: recovery is deterministic, not stochastic.
- **No counterparty risk**: both legs are assumed honoured. No CVA/DVA adjustments.
- **Piecewise-constant hazard interpolation**: the calibrated h(t) has visible jumps at calibration nodes, particularly between 5Y and 7Y where there is no 6Y market quote. This is a structural feature of the interpolation, not market signal — a smoother interpolation (linear in ln Q, or a parametric hazard model) would not exhibit this discontinuity.

## CS01 notes

The CS01 analysis is a bump-and-revalue finite-difference sensitivity. Both parallel and key-rate sensitivities are reported. The sum of key-rate CS01s matches the parallel CS01 to within $322 on $115k of total sensitivity — a 0.3% discrepancy from bootstrap non-linearity, which is the expected order of magnitude.

The notebook documents the caveats: bump definition (market spread vs hazard rate), spread-recovery coupling (held fixed here), and absence of second-order Greeks (gamma).
