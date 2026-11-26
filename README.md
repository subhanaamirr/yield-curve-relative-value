# Yield Curve PCA & Nelson-Siegel Relative Value

Analysis of the US Treasury curve (3M–30Y, March 2006 – November 2026) using principal component analysis and a Nelson-Siegel fair-value model, followed by a backtest of curve butterfly trades on the model's residuals.

**Stack:** Python, pandas, NumPy, SciPy, matplotlib · **Data:** FRED constant-maturity Treasury yields (`DGS3MO` … `DGS30`), 5,189 trading days

---

## Questions

1. **What drives daily curve moves?** PCA on daily yield changes, decomposing them into level, slope and curvature.
2. **Where should each yield be?** A daily Nelson-Siegel fit gives a smooth fair-value curve; the residual (actual − fitted) flags yields that look cheap or rich.
3. **Is the residual tradable?** DV01-neutral and PCA-neutral butterflies, entered when the belly is unusually cheap or rich versus fair value, backtested net of costs with a 2015+ out-of-sample period.

## Method

| Step | Detail |
|---|---|
| PCA | Eigen-decomposition of the covariance of daily yield changes (bp), estimated on 2006–2014 only |
| Nelson-Siegel | Diebold-Li (2006) form with λ fixed; λ chosen by minimising training-sample RMSE (SciPy), then each day fitted by least squares |
| Signal | Butterfly residual vs NS fair value, as a rolling 252-day z-score |
| Rules | Enter at \|z\| > 2, exit at \|z\| < 0.5 or after 60 days; positions set at the close of day *t*, earning P&L from *t* to *t+1* |
| Sizing | $10,000 DV01 on the belly; wings sized 50/50 by DV01, or PCA-neutral (zero exposure to level and slope) |
| Costs | 0.25bp per leg per trade, scaled by DV01 weight |

**Version 2** fixed four weaknesses diagnosed in v1, adding them one at a time so each contribution is visible:
1. Carry and roll-down added to P&L (3M bill as repo proxy; roll along the fitted NS curve)
2. PCA hedge re-estimated monthly on a trailing two-year window
3. Carry filter: only open trades whose carry + roll is favourable
4. Same strategy applied to a 10s20s30s fly

Thresholds, costs and the z-score window were **not** changed between versions.

## Findings

**Curve structure**
- Level, slope and curvature explain **95.5%** of daily yield variance, but level's share (**71.6%**) sits well below the textbook 85–90%. The zero lower bound froze the front end from 2008 to 2015, so level became mainly a long-end factor and slope absorbed the rest (17.7%).
- Nelson-Siegel fits with **7.8bp RMSE**, and its factors track PCA factors closely (daily-change correlations 0.76 / 0.83 / 0.87). Optimal λ = 0.503, placing the curvature hump at 3.6 years.
- The **20Y trades persistently cheap** to fair value: +3.8bp mean residual and a **285-day half-life**, against 43 days for the 5Y. This is consistent with a structural, issuance-driven dislocation rather than noise.

**v1 — price P&L only**
- No out-of-sample edge: 2s5s10s Sharpe −0.13 to +0.05 across three variants; all nine entry/exit settings negative.
- The static PCA hedge was exactly neutral in-sample but picked up level and slope exposure out of sample as factor loadings shifted between rate regimes.

**v2 — with carry, rolling hedge and carry filter**
- 2s5s10s turned marginally positive (out-of-sample Sharpe **0.09**), but the decomposition shows why: **carry + roll +$625k, price P&L −$275k**, costs −$222k. The mean-reversion signal still lost money; the position profited from being paid to hold it, because the 5Y sits at the top of the curve's carry-and-roll profile.
- The rolling hedge **eliminated out-of-sample factor exposure on 10s20s30s** (betas ≈ 0.000), but stayed unstable on 2s5s10s, where the 2Y weight swung from −0.44 to −1.76 because the 2Y barely moved at the zero bound.
- 10s20s30s earned positive price P&L (+$102k) that costs (−$157k) more than offset. The 20Y's cheapness is a persistent level effect, which a rolling-mean z-score is not designed to capture.

**Conclusion**
- No variant is statistically significant (t ≈ 0.3 over 12 years), and every variant loses in **2022**, when the hiking cycle repriced the front end faster than the curve's shape could mean-revert.
- At a one-to-three-month horizon, curve relative value behaves mainly as a **carry trade**, not a mean-reversion trade. I stopped iterating at v2: after diagnosing v1 on 2015+ data, further changes would amount to fitting the test period.

## Limitations

- Constant-maturity yields are interpolated, not tradable bonds; real trades involve specific issues, liquidity and repo specialness.
- Carry is approximate (3M bill as repo proxy, par-bond durations).
- Costs are flat; real bid/ask widens sharply in stress (e.g. March 2020).
- v2 changes were motivated by v1 diagnostics on the 2015+ sample, so that period is not a pristine hold-out for v2.

## Possible extensions

- Trade the 20Y dislocation directly as a carry/level position rather than a mean-reversion signal
- Policy-regime filter using rates-implied hiking/cutting expectations
- Test on specific on-the-run issues with actual repo rates

## Run it

```bash
pip install -r requirements.txt
jupyter notebook yield_curve_pca_nelson_siegel.ipynb
```

The notebook downloads the data from FRED on first run and caches it to `data/`.

## References

- Diebold, F. X. & Li, C. (2006). Forecasting the term structure of government bond yields. *Journal of Econometrics*.
- Litterman, R. & Scheinkman, J. (1991). Common factors affecting bond returns. *Journal of Fixed Income*.
