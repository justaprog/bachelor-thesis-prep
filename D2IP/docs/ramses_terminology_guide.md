# Understanding RAMSeS: A Plain-Language Terminology Guide

*A companion to the RAMSeS research summary · 4 November 2026*

This guide explains the terms used in our summary. Each main concept includes its meaning, an example, and its role in the project. **All small numerical examples are invented for teaching; they are not experimental results.** Implementation references concern the reviewed development fork at [5cdf6a8][fork].

## 1. Start here: data, detectors, and selection

A **time series** is a sequence of observations ordered by time: for example, a temperature measurement every minute.

An **anomaly** is behaviour considered abnormal in the relevant context. A temperature of 80°C could be normal under heavy load and abnormal when a machine is idle.

A **detector** is a model that assigns an anomaly score or decision to an observation or time window. A **selector** decides which detector to use. An **ensemble** combines several detectors.

| Question | Responsible part |
| --- | --- |
| Does this observation look unusual? | Anomaly detector |
| Which detector should we use? | Model-selection method |
| How should several detector outputs be combined? | Ensemble combination method |

An **anomaly score** indicates suspiciousness. Even if normalized to a range such as 0–1, it is not automatically a calibrated probability. A **threshold** converts a score into a decision: flag an anomaly if the score exceeds a chosen value.

A **window** is a short segment of a time series. Window-based processing allows decisions to use recent context instead of treating every timestamp independently.

## 2. Genetic algorithm: searching for a useful combination

A **genetic algorithm (GA)** is a search method inspired by evolution. It creates candidate solutions, evaluates them, and repeatedly modifies promising candidates.

Here, imagine three available detectors: A, B, and C. Candidate solutions are subsets such as `{A, B}` or `{B, C}`.

| GA term | Plain meaning | Example |
| --- | --- | --- |
| Individual | One candidate solution | The subset `{A, C}` |
| Population | A collection of candidates being searched | Several different subsets |
| Fitness | A number measuring candidate quality | Detection quality on validation cases |
| Selection | Choosing candidates to produce the next generation | Prefer subsets with higher fitness |
| Crossover | Combining parts of two candidates | Create a new subset from two parent subsets |
| Mutation | Randomly changing a candidate | Add C or remove A |
| Generation | One round of evaluation and modification | Evaluate, select, and produce new candidates |

A binary representation could encode `{A, C}` as `101`: include A, exclude B, include C.

The search matters because 20 detectors have **1,048,575 nonempty subsets**. Evaluating every subset can be expensive.

**In RAMSeS:** the GA searches detector subsets for the ensemble branch. It does not directly change the neural-network weights of the base detectors. Fitness is the search objective; it is not a separate detector.

**Remember:** a GA searches for a good solution under a budget. It does not generally guarantee the globally best subset.

Background: [DEAP’s evolutionary-algorithm documentation][ga]. Project implementation: [ensemble search][ensemble].

## 3. Stacking and the meta-learner: combining detector outputs

**Stacking** is an ensemble method in which another model learns how to combine the outputs of several base models.

The combining model is the **meta-learner**. “Meta” here means that its inputs come from other models.

Consider one observation:

| Input to the meta-learner | Value |
| --- | ---: |
| Detector A’s anomaly score | 0.90 |
| Detector B’s anomaly score | 0.20 |
| Detector C’s anomaly score | 0.80 |

The meta-learner receives `[0.90, 0.20, 0.80]` and produces the ensemble’s output. It learns a combination rule from training examples rather than always taking the average.

**The two jobs are different:**

| Method | Job |
| --- | --- |
| Genetic algorithm | Decide which detectors belong to the ensemble |
| Stacking meta-learner | Decide how to combine the selected detectors’ outputs |

For example, the GA might choose A and C. The meta-learner then uses their two score columns.

A **random forest**, one supported meta-learner, combines many decision trees. A **decision tree** makes predictions through a sequence of learned conditions. Other supported choices include logistic regression, gradient boosting, and a support vector machine.

Training a meta-learner requires a target signal. Unsupervised base detectors can therefore be part of a selection system that uses synthetic labels for evaluation and combination.

**Remember:** “meta-learner” here means a second-stage combining model. It does not by itself mean learning across datasets or automatically tuning the whole framework.

Background: [scikit-learn’s stacking documentation][stacking]. Project implementation: [ensemble code][ensemble].

## 4. Thompson sampling: choosing while uncertain

**Thompson sampling** is a method for choosing between actions when their quality is uncertain.

It balances:

- **Exploitation:** use an option that appears promising.
- **Exploration:** try alternatives to learn more about them.

Instead of keeping only one fixed quality estimate, the method maintains probability distributions over uncertain quantities. It samples a possible set of values, chooses the action that looks best under that sample, observes feedback, and updates its beliefs.

Illustrative decision:

| Detector | Current estimated average reward | Reward predicted under one sampled parameter set |
| --- | ---: | ---: |
| A | 0.80 | 0.77 |
| B | 0.75 | 0.86 |

B is chosen in this round. Its sampled prediction is higher, even though its current average estimate is lower. Future samples can produce a different choice.

A **reward** is numerical feedback about the chosen action’s quality. A **posterior** is the updated probability distribution after incorporating feedback.

**In RAMSeS:** candidate detectors are the actions, often called “arms.” The framework uses **contextual linear Thompson sampling (LinTS)**: expected reward is modelled as a linear function of a representation of the current window. Context makes the decision depend on the data being processed. The method also includes an **epsilon-greedy** step: with a specified probability, choose a candidate randomly.

**Remember:** Thompson sampling is a selection policy. It needs meaningful feedback; it cannot discover true detection quality without some evaluation signal.

Background: [Russo et al., A Tutorial on Thompson Sampling][thompson]. Project implementation: [Thompson sampling][lints].

## 5. GAN-generated perturbations: constructing challenging test cases

A **perturbation** is a modification to data. For example, changing a measurement from 60.0 to 60.4 introduces a small perturbation.

A **generative adversarial network (GAN)** trains two models against one another:

| Model | General GAN role |
| --- | --- |
| Generator | Produce artificial examples that resemble the training data |
| Discriminator | Distinguish training examples from generated examples |

Their competition helps the generator learn patterns in the data. GAN generation is more structured than simply adding independent random noise.

**“GAN-generated perturbations” therefore means artificial test cases produced using this generative mechanism.** The phrase describes where the cases come from; it does not guarantee that they represent real failures.

Illustrative machine example: pressure and temperature normally follow a relationship. A generated borderline case might resemble ordinary measurements but deviate slightly from that relationship. A detector is then tested on such cases.

**In RAMSeS:** GAN-based testing generates borderline cases to help assess candidate-detector robustness. The paper describes selecting generated points near a discriminator boundary and assigning surrogate labels.

A **surrogate label** is an automatically constructed stand-in for a true annotation. In an ordinary GAN, real-versus-generated discrimination is not the same as normal-versus-anomalous classification.

**Remember:** a generated or borderline case is not automatically a genuine anomaly. This is why generalization from synthetic cases to real anomalies is a research question.

Background: [Goodfellow et al., Generative Adversarial Networks][gan]. Project mechanism: [RAMSeS, Section V-B][paper].

## 6. Univariate UCR series and multivariate server measurements

### Univariate versus multivariate

**Univariate** means one measured variable per timestamp. **Multivariate** means several measured variables per timestamp.

Univariate example:

| Time | Temperature |
| --- | ---: |
| 10:00 | 60 |
| 10:01 | 61 |
| 10:02 | 60 |

Multivariate example:

| Time | CPU use (%) | Memory use (%) | Network traffic (MB/s) |
| --- | ---: | ---: | ---: |
| 10:00 | 25 | 40 | 12 |
| 10:01 | 27 | 41 | 13 |
| 10:02 | 88 | 43 | 1 |

In multivariate detection, relationships between variables can matter as well as individual values.

**Univariate does not mean one data point.** It means one variable measured repeatedly. Many independent univariate series also do not automatically form one multivariate series.

### UCR

**UCR** refers to the University of California, Riverside. Here, it means the **UCR Time Series Anomaly Archive**, a benchmark collection associated with that research group.

Thus, **“univariate UCR series”** means individual one-variable time series from that anomaly archive. UCR is the dataset name, not a detection algorithm.

Use the anomaly archive specifically; the similarly named UCR classification archive serves a different task. Dataset background: [Wu and Keogh’s project page][ucr].

### SMD and SKAB

**SMD** means **Server Machine Dataset**. It contains server-monitoring time series with multiple measurements at each timestamp. The SMD setting described in the RAMSeS paper has 38 variables per machine.

“Server measurements” means recorded system behaviour, such as resource utilization. The three-column table above is an illustration, not a reconstruction of the actual SMD columns. Original dataset source: [OmniAnomaly repository][smd].

**SKAB** means **Skoltech Anomaly Benchmark**. In this project it provides multivariate industrial sensor data associated with water-pump operation.

A **benchmark** is a shared dataset and evaluation setting used to compare methods. Dataset diversity helps test whether a method works beyond one kind of signal. [RAMSeS experimental setup][paper]

## 7. What “key developments in the fork” means

### 7.1 Explaining ensemble membership and detector influence

These answer different questions:

- **Membership:** why did the search include detector A?
- **Influence:** once included, how much does A’s score affect the ensemble output?
- **Ranking contribution:** how much does a particular source ranking affect the consensus?

A detector can be included alongside a similar detector and still have limited influence on particular observations.

### 7.2 SHAP, TreeSHAP, and permutation-based attribution

**SHAP/Shapley attribution** distributes a model output’s difference from a reference among its input features.

For this ensemble explanation, the features are **detector scores**. An attribution to detector A therefore concerns its score’s effect on the combining model; it is not automatically an explanation of an underlying physical fault.

**TreeSHAP** exploits tree structure to compute Shapley attributions efficiently. The fork uses it for random-forest meta-learners.

For other meta-learners, **permutation-based attribution** samples feature orderings and estimates contributions as features are introduced. This replaces enumerating all subsets in the operational explanation path.

This is different from **permutation feature importance**, which shuffles an input feature and measures the resulting change in predictive performance.

The reference choice and attribution method affect the explanation. Attribution alone does not establish causation. [Fork ensemble implementation][ensemble]

### 7.3 Structured facts, LLM narratives, and faithfulness

An **LLM** is a large language model. A **narrative** is its prose explanation.

The project first stores computed facts in an **intermediate representation (IR)**: structured JSON data passed between analysis and narration. The LLM expresses those facts in readable language.

A **faithfulness check** tests whether the narrative remains supported by the recorded facts. A **hallucination** is an unsupported claim; an **omission** is missing required information. The implemented checks are mechanical and have documented limitations. [IR and verifier][explanations]

### 7.4 Nearly equivalent ensembles and trial variability

Two ensembles might score 0.850 and 0.849. A tiny difference may be smaller than variation caused by fitting the combining model again.

**Nearly equivalent ensembles** are candidates close enough in measured quality that choosing one should not be interpreted as proving it uniquely superior.

A **Monte Carlo trial** repeats an experiment with a new random draw, such as noise added to measurements. **Trial variability** describes how scores or rankings change across those repeats. Stable average performance and consistently winning individual trials are different findings. [Ensemble analysis][ensemble] · [Monte Carlo implementation][mc]

### 7.5 Combined metrics and configurable meta-learners

A **combined metric** merges quality measures into one objective. For example, equal weighting of F1 = 0.8 and PR-AUC = 0.6 gives fitness = 0.7.

In the fork, each robustness test produces one detector ranking using the configured objective. **Rank aggregation** then combines several such rankings into a consensus. Different objectives or aggregation rules can produce different winners.

A **configurable meta-learner** lets the researcher choose the combining model for a run. This does not mean that the system automatically chooses its best configuration. [Pipeline][pipeline]

### 7.6 Excluding ground-truth annotation columns

**Ground truth** is the reference annotation used to evaluate predictions. In SKAB, the `anomaly` and `changepoint` columns describe the answer rather than ordinary sensor inputs.

Allowing such columns into detector input can cause **target leakage**: information about the answer influences predictions. The fork explicitly excludes both columns from features.

This explains the importance of the loading correction; it does not establish which data-processing revision generated the paper’s published results. [Data loader][loader]

## 8. Other useful terms from the summary

| Term | Plain meaning |
| --- | --- |
| Precision | Of the items flagged, how many were truly anomalous? |
| Recall | Of the true anomalies, how many were found? |
| F1 | Harmonic mean of precision and recall; balances the two |
| PR-AUC | Area under a precision–recall curve obtained by varying the decision threshold |
| Robustness | Maintaining useful performance under specified disturbances |
| Generalization | Performing well on relevant data beyond the cases used for development |
| Hyperparameter | A setting chosen outside ordinary model fitting, such as a search budget |
| Ablation | Removing or replacing a component to measure its added value |
| Offline | A preparation/selection phase before the evaluated stream |
| Online | Processing observations as they arrive |
| Re-optimization | Repeating a selection or optimization procedure |
| Distribution shift | A change in the statistical properties of the data |
| Synthetic-to-real transfer | Whether learning or selection on artificial cases helps on real cases |

**Metric caution:** the F1 explanation above is introductory. Time-series evaluation can use event- or range-based rules; use the same protocol when comparing reported numbers.

## 9. Re-read the summary using these translations

| Original wording | Plain-language translation |
| --- | --- |
| Genetic algorithm with a stacking meta-learner | Search for a useful detector subset, then train a model to combine its scores |
| Contextual linear Thompson sampling | Choose detectors using uncertain performance estimates that depend on the current window |
| GAN-generated perturbations | Artificial, structured cases used to challenge detectors |
| Univariate UCR series | One-variable time series from the UCR anomaly benchmark |
| Multivariate server measurements | Several measurements recorded over time for a server |
| Key developments in the fork | Significant changes in a separately maintained project version |

[paper]: https://arxiv.org/pdf/2602.21766
[fork]: https://github.com/hcoskunn/RAMSeS/tree/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef
[ga]: https://deap.readthedocs.io/en/master/api/algo.html
[stacking]: https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.StackingClassifier.html
[thompson]: https://arxiv.org/abs/1707.02038
[gan]: https://arxiv.org/abs/1406.2661
[ucr]: https://wu.renjie.im/research/anomaly-benchmarks-are-flawed/
[smd]: https://github.com/NetManAIOps/OmniAnomaly
[ensemble]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Metrics/Ensemble_GA.py
[lints]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Model_Selection/Thompson_Sampling.py
[pipeline]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/app.py
[explanations]: https://github.com/hcoskunn/RAMSeS/tree/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Explainability
[mc]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Model_Selection/Sensitivity_robustness/Monte_Carlo_Simulation.py
[loader]: https://github.com/hcoskunn/RAMSeS/blob/5cdf6a8bd1c8b1fd9989ac127e028f860f290bef/Datasets/load.py

