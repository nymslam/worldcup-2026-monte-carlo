# Methodology

This document explains the modelling decisions behind the 2026 World Cup forecast, the
reasoning for each, and how the model was validated. The emphasis throughout is on
*defensible* choices and on measuring the model rather than trusting it because the
output looks right.

## 1. The match model: Dixon–Coles

Every forecast ultimately rests on a model of a single match. Rather than predicting
win/draw/loss directly, the model predicts **goals**, using a Dixon–Coles model
(Dixon & Coles, 1997; building on Maher, 1982).

For a match between team *i* and team *j*, the expected goals are

```
log λ_i = a_i − d_j + h·(home)
log λ_j = a_j − d_i
```

where `a` is a team's attack strength, `d` its defence strength, and `h` a home-advantage
term applied only when a team genuinely plays at home (the dataset's `neutral` flag
controls this). The number of goals each team scores is then drawn from a Poisson
distribution, with the Dixon–Coles low-score correction `τ` adjusting the probabilities
of 0–0, 1–0, 0–1, and 1–1 results, which independent Poisson models get wrong.

The parameters are estimated by **weighted maximum likelihood**. Attack strengths are
anchored (mean zero) for identifiability; defence strengths follow from the data.

## 2. Which matches, and how much each counts

Two design choices shape the training data.

**Friendlies are excluded as primary signal but kept as bridges.** Friendlies are noisy
— teams rest starters, experiment with line-ups, and play at low intensity. The initial
instinct was to drop them entirely and use only competitive matches. That turned out to
be a mistake (see §3), so the final model keeps them at a low weight.

**Each match is weighted by recency and importance:**

```
weight = competition_importance × exp(−ln2 · age_in_days / HALF_LIFE)
```

The time-decay half-life is roughly 2.5 years, reflecting that international squads turn
over slowly but a result from a decade ago says little about the current team.
Competition importance is tiered: World Cup matches count most, then continental finals,
then qualifiers and Nations League, then friendlies at a low bridging weight. Blowout
scorelines are capped (a 7–0 over a minnow counts the same as a 5–0) so that thrashings
of weak opponents don't inflate attack ratings.

## 3. The confederation-bias problem

The most important finding of the project came from inspecting the first set of ratings,
which were clearly wrong: New Zealand above the Netherlands, Brazil 13th, Asian and
Oceanian sides crowding the top.

This was not a coding bug. A Dixon–Coles model can only compare two teams by tracing the
chain of matches connecting them. Competitive matches are almost entirely *intra*-confederation
— qualifiers and continental cups are played within Asia, within Europe, within South
America. The matches that bridge confederations are mostly friendlies. By excluding
friendlies, the model was left with six weakly-connected islands and no way to calibrate
their relative levels. Each confederation's internal order was fine, but the islands
floated to arbitrary heights: confederations whose strong teams pile up wins against weak
regional opponents drifted upward, while South America — where every match is against a
strong side — was pushed down.

The fix follows directly from the diagnosis: reintroduce friendlies, but only at a low
weight (a tunable `FRIENDLY_WEIGHT`, default 0.20), so they supply the cross-confederation
links without their noise overwhelming the competitive results. After the fix, Argentina
rose to first, Brazil to fourth, and the South American sides returned. The effect was
confirmed on synthetic data with deliberately isolated confederations: with the bridges
removed the model could not distinguish a stronger confederation from a weaker one; with
them restored it recovered the true ordering.

## 4. The Elo prior for thin teams

A 48-team field includes debutants and rarely-seen sides (e.g. Curaçao, Cape Verde) with
little or no competitive history — too little for Dixon–Coles to estimate stable
parameters. To handle this, a self-computed Elo rating serves as a **Gaussian prior**:
the fitting objective penalises each team's overall strength (`a + d`) for departing from
its Elo-implied level. Teams with abundant data have a sharply peaked likelihood that
overrides the prior; data-thin teams have a flat likelihood, so they shrink toward their
Elo strength. The shrinkage is automatic and proportional to data scarcity, with no need
to hand-flag which teams are "thin." The same Elo also serves as a baseline in the
backtest (§7).

## 5. Host advantage

The three hosts (USA, Canada, Mexico) receive a home-advantage term, but a *scaled-down*
one (`HOST_ADVANTAGE_SCALE`, default 0.40 of the fitted home edge). Two reasons: a World
Cup host plays in large, largely neutral stadiums rather than a true home ground, and the
advantage compounds across up to eight matches, so applying the full term over a deep run
overstates it badly. An early version without this scaling gave the hosts implausibly high
title chances; scaling it back brought them into line with the market's view that home
advantage counts for little here.

## 6. Simulating the tournament

The full 2026 format is simulated: 12 groups of four, with the top two from each group
plus the eight best third-placed teams advancing to a Round of 32, then a Round of 16,
quarter-finals, semi-finals, and final. Group tiebreakers use points, then goal difference,
then goals scored, then a random draw (the head-to-head and fair-play tiebreakers are not
modelled). Knockout draws are resolved with a short extra-time period and then a coin-flip
penalty shootout.

The Round-of-32 bracket uses a strength-seeded approximation of FIFA's fixed positional
template and its 495-combination third-place matrix. This is a deliberate simplification:
it preserves the bracket topology and seeding spirit, and its effect on title and stage
probabilities is second-order, since the strongest teams advance regardless of which exact
slot they fill. The tournament is simulated 20,000+ times and the results aggregated into
each team's probability of reaching every stage.

## 7. Validation

Plausible-looking output is not evidence. The model is validated two ways, both built on
a strict **as-of refit**: to predict a past tournament, the model is fit using only matches
from before that tournament's kickoff, with the time-decay and Elo prior both anchored to
that date. This prevents future information from leaking into the past.

**Match level.** For the 2018 and 2022 World Cups, the model predicts the actual matches
(which live in the dataset) and is scored with **Ranked Probability Score** (the standard
metric for ordinal win/draw/loss outcomes) and log loss, against two baselines: an Elo-only
model and a naive base-rate. Beating the base-rate confirms the model does something;
matching or beating Elo confirms the extra machinery earns its complexity. A reliability
check (Expected Calibration Error) confirms the probabilities are honest.

**Stage level.** The tournaments are simulated in their historical 32-team format, and each
team's predicted probability of reaching each round is compared to what actually happened.
Pooling these (team × stage) events across both tournaments gives enough points for a
calibration curve on the simulation engine itself.

**Hyperparameter tuning.** Values like `FRIENDLY_WEIGHT` are chosen by backtesting on *one*
tournament and reporting final results on the *other*, never tuning and reporting on the
same data.

## 8. Limitations

- **Results, not squads.** The model sees only scorelines, so it underrates talent-rich
  sides whose recent results are unspectacular — France is the clearest example. This is a
  fundamental limitation of scoreline-based models; the principled fix is a squad-value or
  market-odds prior.
- **Residual confederation lean.** The bridge fix is partial; a mild upward lean for Asian
  sides remains, tunable via the friendly weight or an external Elo prior.
- **Approximated bracket.** The exact FIFA third-place matrix is not reproduced.
- **Thin calibration data.** Only two tournaments are available for stage-level validation,
  so it is directional rather than definitive.

## References

- Dixon, M. & Coles, S. (1997). *Modelling Association Football Scores and Inefficiencies
  in the Football Betting Market.* Journal of the Royal Statistical Society: Series C, 46(2).
- Maher, M. J. (1982). *Modelling Association Football Scores.* Statistica Neerlandica, 36(3).
