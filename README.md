# 🏎️🏁 Fantasy F1 Optimizer 

**Separating driver skill from car performance in Formula 1 — and using it to pick fantasy teams.**

![Status](https://img.shields.io/badge/status-in%20development-orange?style=flat-square)
![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![FastF1](https://img.shields.io/badge/data-FastF1-E10600?style=flat-square)

> **Work in progress.** This repository documents an active project. The design and methodology below are settled; implementation is underway. Nothing here reports results yet — findings will be added as each phase is validated.

---

## The Question

When Verstappen wins, how much of that is him and how much is the car?

It's the argument every F1 fan has had, and it's genuinely hard to answer: a driver and their car only ever appear together. You never observe one without the other. Any attempt to rank drivers by raw results is really ranking machinery.

This project estimates the two effects separately, then uses them to answer a practical question — **which driver-and-team combination will score best at the race coming up, given the circuit and the weather?**

## The Method

The sport has a natural experiment built into it: **teammates drive identical cars.**

Any gap between two teammates is therefore pure driver — the machinery cancels out. A simplified example:

| Team | Driver | Points |
|---|---|---|
| McLaren | Norris | 20 |
| McLaren | Piastri | 16 |
| Ferrari | Leclerc | 18 |
| Ferrari | Hamilton | 12 |

- **Car effect** = team average → McLaren 18, Ferrari 15
- **Driver effect** = deviation from teammate average → Norris +2, Piastri −2, Leclerc +3, Hamilton −3

Once the two are separated, combinations that have never existed become estimable. Leclerc in a McLaren = 18 (car) + 3 (driver) = **21 points**, despite no such pairing ever having raced.

Teammate pairs link drivers *within* a team; drivers who transfer between teams link those teams to each other. Chained together, these comparisons connect the entire grid into a single estimable network — which is why the analysis spans multiple seasons rather than one.

The production estimator is a regression of the form `points ~ driver + constructor`, fitted across every driver-race in the dataset. This uses the full network simultaneously rather than treating each team in isolation, which matters when a team happens to field two unusually strong or weak drivers.

## Conditioning on context

Performance isn't constant. Monaco rewards qualifying because overtaking is near-impossible; Monza rewards straight-line speed; rain reshuffles everything.

So effects are estimated **per context**, not globally. The obstacle is sample size — Monaco happens once a year, which is not enough to learn from. The fix is to group circuits by type rather than by name:

| Archetype | Circuits |
|---|---|
| Street | Monaco, Singapore, Baku, Jeddah |
| High-downforce | Hungary, Zandvoort, Barcelona |
| Power-sensitive | Monza, Spa, Las Vegas |
| Stop-start | Montreal, Mexico City |

A single street circuit is n=1. The street *archetype* is n=8 across two seasons. Weather is bucketed on the same principle.

These groupings are a modelling decision with no objectively correct answer, and they are the part of this project most open to challenge — which is why they're stated explicitly rather than buried in code.

## Intended Output

Fantasy teams are selected *before* a race, when the weather is still a forecast. Rather than collapsing that uncertainty into a single answer, the tool reports both scenarios:

```
MONZA — DRY                                      cost    proj.
1. Verstappen + Norris + McLaren                 $81M    104 pts
2. Leclerc + Verstappen + Ferrari                $79M     99 pts
3. Norris + Piastri + McLaren                    $78M     96 pts

MONZA — WET                                      cost    proj.
1. Verstappen + Alonso + Red Bull                $72M    108 pts
2. ...
```

Any two drivers may be paired with any constructor, giving 231 driver pairs × 11 teams = **2,541 combinations**, ranked under the $100M cost cap.

## Roadmap

- [ ] **1 · Foundation** — FastF1 ingestion, fantasy scoring engine, 2024–25 seasons loaded and cleaned
- [ ] **2 · Conditioning** — circuit archetypes, weather buckets, context-specific baselines
- [ ] **3 · Decomposition** — separate driver and constructor effects; generate and rank combinations
- [ ] **4 · Validation** — backtest across 2024–25 against a naive baseline
- [ ] **5 · Extension** — test whether driver × car interaction terms improve on the additive model; interactive demo and written analysis

## Design notes

**Scoring rules are isolated.** Fantasy scoring changes between seasons, so every rule — position points, overtake bonuses, penalties, budget, roster size — lives in a single configuration module. Adapting to a rules change is a one-file edit.

**Regulation changes are handled asymmetrically.** 2026 brought new power units, revised aerodynamics, and an eleventh team. Driver effects are skills and survive a regulation reset; car effects belong to a specific car and do not. Driver estimates therefore draw on historical seasons, while constructor estimates are re-fitted from current-season results and weighted more heavily as races accumulate.

**The budget is a filter, not the objective.** With three selections, most combinations fall comfortably under the cap. It constrains only the top end. This project is about estimation, not constrained optimization.

## Known Limitations

Stated upfront, because they shape how the output should be read:

- **Early-season estimates are noisy.** Constructor effects re-fit from a handful of races carry wide uncertainty, and the tool should communicate that rather than imply false precision.
- **Interaction effects may not be supportable.** The hypothesis that specific drivers suit specific car characteristics is plausible and harder to establish than the additive model. Two seasons may not contain enough driver-team variation to test it. A negative result here is a finding, not a failure.
- **Fantasy points are a proxy for performance.** The scoring system rewards some things disproportionately — grid position changes, for instance — so a driver effect estimated on fantasy points is not identical to one estimated on pure pace.
- **Retirements are partly outside driver control.** Mechanical failures attach to results that the model attributes to a driver-car pair, adding noise that two seasons may not fully average out.

## Stack

Python · pandas · NumPy · statsmodels · FastF1 · Matplotlib

---

Built by [Bhumika Kalra](https://github.com/bkalra04) · MIT Licensed
