# ST540 Final Project — Research Proposal

## Research Question

**Are tournament rounds independent draws from a player's skill distribution (random walk), or does round-to-round performance within a tournament persist in a way that warrants a Markov chain structure?**

Golf commentary constantly invokes within-tournament momentum: "once he got going, he rode that momentum all the way to Sunday." The statistical null is that each round is an independent draw from that player's skill distribution, adjusted for shared course-day conditions. We test which structure the data supports, separately by strokes-gained component to identify where (if anywhere) momentum is a real phenomenon.

---

## Why This Question

The hot-hand question is well-known from basketball (Gilovich, Tversky, Vallone 1985; Miller & Sanjurjo 2018), where the consensus is that apparent streaks are statistically indistinguishable from independent draws. Whether the same holds in golf — where rounds are separated by ~24 hours and involve different psychological and physical setups than consecutive shots in basketball — is much less studied. Most importantly, we scope the question to within a single tournament, where conditions are held constant (same course, same week, same player). If momentum exists anywhere in golf, it's here.

The question maps cleanly onto two canonical statistical models:
- **Independence / random walk around baseline**: rounds are iid conditional on player skill and round-level course conditions
- **Markov chain / AR(1) structure**: each round's residual depends on the prior round's residual via a persistence parameter ρ

Bayesian model comparison between these specifications is a textbook application of the methodology and produces a single number (the posterior on ρ, or the Bayes factor) as the headline result.

---

## Models

### Model A — Independence (null)

For player *i*, tournament *t*, round *r*, SG component *k*:

```
sg_itrk = alpha_ik + round_effect_trk + epsilon_itrk
epsilon_itrk ~ Normal(0, sigma_k)
alpha_ik ~ Normal(mu_k, tau_k)   # player skill, hierarchical
round_effect_trk ~ Normal(0, sigma_round_k)
```

Rounds within a tournament are conditionally independent given player skill and shared round conditions.

### Model B — Markov (alternative)

```
sg_itrk = alpha_ik + round_effect_trk + rho_k * epsilon_it(r-1)k + eta_itrk
eta_itrk ~ Normal(0, sigma_k)
rho_k ~ Normal(0, 0.5)   # persistence parameter, component-specific
```

Each round's residual is a weighted function of the prior round's residual. `rho_k` is the parameter of interest: the posterior on `rho_k` directly tests whether within-tournament momentum exists in component *k*.

### Hierarchical extension

Per-player persistence: `rho_ik ~ Normal(rho_k_pop, tau_rho_k)`, partial pooling across players. This tests whether momentum is a universal population effect or varies by player — addressing the "some players are streaky, others aren't" folk claim.

### Priors

- `mu_k ~ Normal(0, 1)` — population mean skill per component
- `tau_k ~ HalfNormal(0.5)` — between-player variation
- `sigma_k ~ HalfNormal(2)` — within-round noise
- `sigma_round_k ~ HalfNormal(1)` — course-day effect magnitude
- `rho_k ~ Normal(0, 0.5)` — weakly informative, allowing positive, negative, or zero persistence
- `tau_rho_k ~ HalfNormal(0.3)` — between-player variation in persistence

Plan for a prior sensitivity slide: run the model with tighter and looser priors on `rho_k` and `tau_rho_k` to show robustness of the headline result.

---

## Data

- Source: PGA Tour round-level strokes-gained data, stored in local SQLite database (`golf.db`)
- Table: `rounds`, columns `sg_ott`, `sg_app`, `sg_arg`, `sg_putt`, `sg_total`
- Coverage: 2019-2025, PGA Tour only (tour='pga')
- Filter: players with at least N rounds for inclusion in hierarchical player-effect estimation (threshold to be set based on posterior behavior — start with ~40)
- Tournament structure: rounds 1-4, made-cut players contribute all 4 rounds, missed-cut contribute 2
- Observation count: ~200K round-player-component observations across ~2000 players and ~300 tournaments

### Conditioning choices to document

1. **Residual definition**: residuals computed as `sg_itrk - (alpha_ik + round_effect_trk)` from the fitted model, not pre-computed. The round effect absorbs shared course-day conditions so that residuals are player-specific deviations.
2. **Cut treatment**: missed-cut players contribute 2 rounds, which still provides one round-pair for the Markov structure (R1 → R2). Not dropping them avoids selection bias toward players who played well enough to make cuts.
3. **Component analysis**: run the model separately for each SG component (OTT, APP, ARG, PUTT) to test whether momentum is component-specific.

---

## Methods

- Hierarchical Bayesian model via PyMC
- Non-centered parameterization for player skill and persistence parameters to aid convergence
- 4 chains, 2000 draws, 2000 tuning steps
- NUTS sampler with target_accept=0.9
- Convergence diagnostics: R-hat < 1.01, effective sample size > 400 per parameter
- Model comparison: LOO-CV (leave-one-out cross-validation via Pareto-smoothed importance sampling) to compare Model A vs Model B, per component

---

## Expected Results and Headline Figures

### Primary headline

Posterior density of `rho_k` for each of the four SG components, overlaid on a single plot with credible intervals marked. Interpretation:
- If `rho_putt` has posterior mass clearly above zero while `rho_ott` is centered on zero, within-tournament putting momentum is real and component-specific
- If all four posteriors overlap zero, momentum is a narrative artifact
- If all four are uniformly positive, momentum is a general phenomenon

### Secondary figures

1. **LOO-CV comparison** between Model A and Model B per component, with standard errors, showing which model the data prefers
2. **Per-player shrinkage plot** for `rho_ik` in the component with the strongest effect, demonstrating hierarchical pooling (thin-data players shrunk toward population, thick-data players retain their individual estimates)
3. **Prior sensitivity slide**: posterior of `rho_k` under three different prior specifications to show robustness

---

## Slide Structure (target: 10 slides)

1. **Cover**: title, author, course
2. **Motivation**: momentum narrative in golf commentary, contrast with basketball hot-hand literature, why within-tournament is the right scope
3. **Research gap**: prior work on golf streakiness has used autocorrelation coefficients descriptively; we pose it as explicit Bayesian model comparison with hierarchical structure
4. **Data**: PGA Tour 2019-2025, SG at round level, panel structure, sample sizes
5. **Models**: Model A (independence) and Model B (Markov) equations side by side
6. **Priors**: specification and justification
7. **Results A**: posterior density of `rho_k` for four components on one plot
8. **Results B**: LOO-CV comparison table, per-player shrinkage plot for headline component
9. **Robustness**: prior sensitivity plot
10. **Conclusions**: what the posterior says about momentum, strengths, limitations, caveats about within-tournament vs cross-tournament scope

---

## Strengths

- Methodology is recognizably graduate-level Bayesian: hierarchical model comparison, partial pooling, LOO-CV, prior sensitivity
- Clean binary between two canonical model structures (random walk vs Markov), easy to explain in a stats class
- Single-number headline result per component (posterior on ρ) with credible interval
- Non-trivial result in either direction: confirming null is interesting given folk wisdom; confirming momentum overturns the basketball precedent
- Data already in database, no external scraping required

---

## Weaknesses / Limitations (for the final slide)

- Within-tournament scope limits generalization; cross-tournament persistence is a separate question
- SG metrics are themselves estimated against the field, so correlated measurement error across rounds within a tournament could induce spurious persistence
- Missed-cut players contribute only one round pair, potentially biasing the Markov estimate toward made-cut dynamics
- Course-day effects absorbed at the tournament-round level may not fully capture individual pairing/tee-time effects (morning vs afternoon waves)

---

## Working Environment

- Project directory: `~/Desktop/golf-trading/`
- Database: `./golf.db` (SQLite)
- Conda environment: `golf-pymc` for PyMC work
- Relevant existing infrastructure:
  - `rounds` table with SG components already populated
  - `round_fixed_effects` table with per-round (event, year, tour, round_num) fixed effects already estimated
  - `adjusted_sg_snapshots` for player skill estimates (Ridge-based, can serve as informed priors if desired)

Existing round fixed effects from `round_fixed_effects` can be used directly to define residuals, avoiding re-estimation. Player skill priors can be informed by `adjusted_sg_snapshots` values for each player at the start of each tournament.