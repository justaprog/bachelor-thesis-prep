# Ramses Example

This example helps going through Ramses high-level architecture step by step.

## Data:
Time Temperature Pressure
1       20          10
2       22          11
3       18          9
4       20          10
5       22          11
6       20          10

## Synthetic validation cases (Temperature, Pressure, label, changed):
V1: (34, 10, anomaly, temperature spike)
V2: (20, 4, anomaly, pressure drop)
V3: (22, 11, normal, no change)
V4: (26, 10, anomaly, smaller temp deviation)

## 3 detectors: 

A: Temperature detector
B: Pressure detector
C: relationship detector

## Detector Rule:
- A: Flag (=1) if temp > 25
- B: Flag if pressure < 8
- C: Flag if |temp -2*pressure| > 10

Note: C as a additional feature to capture the relationship between temperature and pressure

## Detector outputs:

- V1: A=1, B=0, C=1
- V2: A=0, B=1, C=1
- V3: A=0, B=0, C=0
- V4: A=1, B=0, C=0

Observations:
- Flagged normal when all outputs = 0
- Only B or C = 1 not sure
-> Need more data to train the meta learner

## Evaluation detectors: Precision, Recall, F1-score (not by hand) on validation set

Tuple (Pre, Recall)

A: (1, 2/3)
B: (1, 1/3)
C: (1, 2/3)

- Observations:
A and C better > B

- How about combining (using OR)?:
e.g: A,C -> (1, 1) -> better than individual detectors

other subsets: {A,B} -> (1,1)

## Branch: Ensemble selection: Genetic algorithm and stacking meta learner

- Question: Which subset to choose and how to combine 
-> GA select subset
-> stacking meta learner combines outputs

## Branch: Single-model selection

- Question: Which single detector to choose as the final model?
-> Thompson sampling -> ranking
-> robustness tests: GAN, threshold sentivity, Monte Carlo simulations -> more rankings
-> Rank aggregation-> final ranking of single detectors

## Evaluation on separate, unseen labeled data

- ensemble vs single-model selection

## Questions

- Why synthetic data? possibly: small labeled dataset, more edge cases, distribution shift
- How to generate synthetic data? possibly: samples, GAN perturbations (apparently 
only for robustness test -> how about validation set?: rule-based)
- When to update because of distribution shift? Reoptimization introduces computational overhead
during deployment -> Open question: event-based updates -> possibly: distribution shift detection, 
accuracy degradation (threshold) -> What does the reoptimization look like? 
Does it solve the distribution shift problem or improve accuracy under new conditions?
What might be the reason for accuracy degradation other than distribution shift?
Domain-based? -> Add rules for domain-specific knowledge -> A lot more questions to solve this problem...


