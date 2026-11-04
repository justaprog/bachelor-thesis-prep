# RAMSeS: Paper and Implementation Summary

*Research notes for a potential bachelor’s thesis · Reviewed 4 November 2026*

**Scope:** [RAMSeS paper][paper], upstream revision [8e6d61d][upstream], and development fork [5cdf6a8][fork]. Paper results, implementation findings, and proposed research questions are distinguished below.

## 1. Problem

Anomaly detectors perform differently across time-series datasets and operating conditions. Changes in normal behaviour can also make an earlier model choice unsuitable. RAMSeS investigates how to select an effective detector or ensemble and adapt that choice over time. [Paper][paper]

## 2. Methods and approaches

### Core framework

RAMSeS starts with a pool of candidate detectors and follows two complementary branches:

| Branch | Approach | Output |
| --- | --- | --- |
| Ensemble selection | A genetic algorithm searches detector subsets; a stacking meta-learner combines their scores | A selected ensemble |
| Single-model selection | Contextual linear Thompson sampling with random exploration, combined with robustness rankings | A ranked detector pool and selected detector |

Robustness testing uses GAN-generated perturbations, near-threshold statistical cases, and Monte Carlo noise trials. Markov-based rank aggregation combines the rankings. Synthetic anomaly injection supplies labelled evaluation cases; online processing periodically repeats selection. [Implementation][pipeline]

### Current implementation and fork

The repositories provide data loading, detector training/checkpoint loading, selection, reporting, and a local web interface.

The fork contains **24 commits beyond the inspected upstream revision**, changing 43 files. Key developments include:

- Explanations of ensemble membership, detector influence, and ranking contributions.
- Structured facts converted into LLM narratives, with mechanical faithfulness checks.
- TreeSHAP for random-forest meta-learners and permutation-based attribution for other meta-learners, replacing exhaustive attribution in the operational path.
- Analysis of nearly equivalent ensembles and Monte Carlo trial variability.
- One combined-metric ranking per robustness test and configurable meta-learner choice.
- A SKAB loading correction that excludes both ground-truth annotation columns from input features.

These changes extend the system and can affect its outputs; their performance benefits require evaluation. [Fork][fork] · [Ensemble code][ensemble]

## 3. Experiments and results

### Results reported in the paper

The evaluation covers univariate UCR series, multivariate server measurements (SMD), and industrial sensor data (SKAB). Table III reports:

| Dataset | RAMSeS average F1 | UMS average F1 |
| --- | ---: | ---: |
| UCR | 0.83 | 0.66 |
| SMD | 0.89 | 0.80 |
| SKAB | 0.86 | 0.86 |

RAMSeS leads the compared methods on UCR and SMD and ties UMS on SKAB. Online experiments report an accuracy–resource trade-off when re-optimization is enabled. These findings apply to the paper’s benchmark protocol and configurations. [Paper, Section VI][paper]

### What our repository review established

Source inspection confirmed the features described above. Two fork test modules covering structured explanations and LLM/verifier behaviour passed **137 tests**. We did **not** reproduce the complete pipeline or benchmark results. The exact revision used for the paper’s experiments remains unconfirmed.

## 4. Limitations and further research

### Stated limitations versus current implementation

| Paper limitation / future direction | Status in inspected code |
| --- | --- |
| Online update overhead; trigger updates when needed | Background updates and skipped GAN execution reduce some overhead; updates remain periodic |
| System configuration; transfer useful settings across datasets | Fixed settings and detector-level tuning exist; no transfer-based system configuration identified |
| Numerical data scope; extend to categorical, mixed-type, or event sequences | No dedicated implementation identified |
| Interpretability; add SHAP or LLM explanations | Substantial implementation exists, especially in the fork; reliability and usefulness remain evaluation questions |

Source: [Paper, Section VII][paper], compared with the reviewed revisions.

### Prioritized research proposals

The following are **our proposed directions**, not confirmed available thesis topics or established novelty claims.

| Priority | Research question | Bounded contribution and evaluation |
| --- | --- | --- |
| 1. Change-triggered re-selection | Can a trigger using no real anomaly labels reduce update cost while preserving detection quality? | Implement one trigger; compare fixed, periodic, and triggered selection using accuracy, runtime, update count, and adaptation delay |
| 2. Synthetic-to-real generalization | Which synthetic anomaly settings select detectors that perform well on unseen real anomalies? | Fix the detector pool; compare a few injection strategies using selection quality and stability across seeds |
| 3. Selective component execution | Can expensive selection tests run only when cheaper evidence is inconclusive? | Evaluate component contributions, then one conditional execution policy using detection quality and computational cost |

**Experimental prerequisite:** distinguish synthetic labels, permitted human feedback, and held-out real labels. Keep selection, threshold tuning, and final evaluation separate; compare methods with the same detector pool and documented budgets.

**Thesis preference:** direction 1 offers a direct connection to the paper’s future work and a concrete implementation point. Direction 2 suits a stronger focus on machine-learning assumptions. Existing explainability work should be discussed with the chair before proposing overlapping extensions.

[paper]: https://arxiv.org/pdf/2602.21766
[upstream]: https://github.com/Maxoud99/RAMSeS/tree/8e6d61d5748350c872e220c400a7d3c4cf0ea650
[fork]: https://github.com/hcoskunn/RAMSeS/tree/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef
[pipeline]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/app.py
[ensemble]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Metrics/Ensemble_GA.py

