# Round 1 Research Note — The Sequential Matching Problem

**Proposed approach:** Constraint-first sequential allocation with targeted clarification  
**Release:** Public synthetic release 1.0.0  
**Status:** Preliminary implementation and evaluation; findings are limited to the runs reported below.

## 1. Problem interpretation

The policy makes two sequential decisions on each simulated day:

1. **Clarification:** choose which unknown profile fields to ask about under a limited daily budget.
2. **Allocation:** choose a batch of introductions among currently available members.

The policy must act only on information available in the current request and its own persistent memory. It must not use future arrivals, hidden simulator traits, private response propensities, or outcomes that have not yet become observable.

An introduction is valid only when the official reciprocal hard-constraint check returns `feasible`, both members are available, the pair has not previously been introduced, and neither member is used more than once in the same batch. A known hard-constraint conflict makes a pair infeasible; missing hard information requires clarification rather than being treated as approval. A soft compatibility score must never override hard feasibility.

The primary objective is **mutual second-meeting intention (MSMI) per 100 arrived members**. Coverage and mutual acceptance are useful secondary outcomes; clarification cost and inference time also matter. Missing or delayed feedback is not automatically a negative outcome.

## 2. Research hypothesis

Our hypothesis is that a constraint-first policy that combines (a) official reciprocal feasibility checks, (b) targeted clarification of missing hard constraints, and (c) compatibility-weighted non-overlapping allocation can improve allocation quality or coverage relative to simple baselines, especially when profiles are incomplete.

This is a hypothesis to test, not an established result. In particular, higher mutual acceptance or coverage does not necessarily imply higher MSMI. The policy should therefore be selected primarily on the benchmark's MSMI objective, with other metrics used to understand trade-offs.

## 3. Proposed policy

### 3.1 Feasibility-first candidate graph

For each decision, consider only currently available members and exclude previously introduced pairs. Use the release's `eligibility(a, b)` function to classify candidate pairs. Retain only edges marked `feasible` for allocation. Pairs marked `needs_clarification` may inform question selection, but must not be introduced until the official checker returns `feasible`.

This is important because the public release is highly constrained: in a pairwise scan of the public profile snapshot, only about 0.5% of possible pairs were immediately feasible, while about 12.9% required clarification. These figures describe that public snapshot and should not be assumed to hold in every generated evaluation episode.

### 3.2 Compatibility scoring

The initial heuristic scores feasible pairs using observable soft-field agreement. It emphasizes `relationship_goal`, `relationship_pace`, `lifestyle`, and `conversations`, with smaller tie-breaking weights for other available soft fields. The current weights are engineering hypotheses, not fitted parameters or real-world relationship estimates.

Unknown values are treated as unknown: they are not silently imputed as mismatches or as positive evidence. The heuristic uses deterministic tie-breaking to make results reproducible.

### 3.3 Non-overlapping allocation

The policy first builds a non-overlapping batch by considering high-utility feasible edges. It then attempts local two-edge replacements involving the same four members when the replacement increases total edge utility.

This is a local-improvement heuristic, **not an exact maximum-weight matching solver**. A future ablation should test whether a stronger global optimizer improves MSMI enough to justify its added complexity and runtime.

## 4. Clarification strategy

The daily clarification budget is 12 units. A `constraints` bundle costs 3 units; a named soft-field question costs 1 unit. The initial policy prioritizes available members with unresolved, non-declined hard fields, using the number of currently uncertain candidate edges as a simple estimate of how broadly an answer could help. It then uses remaining budget for selected soft fields among members who have feasible alternatives.

The policy avoids repeatedly asking a hard bundle when a hard field has already been declined. A declined answer remains unknown and is not inferred indirectly. This strategy is a practical first heuristic rather than a formal value-of-information solution. Its main risk is that it may spend budget on information that later proves less useful than expected.

## 5. Missingness, feedback, and evaluation validity

The data contract distinguishes observed, not-asked, and declined fields. `null` means unknown, not zero or false. The policy uses only the current observable state and policy-owned memory.

Historical introduction records with null propensity do not support inverse-propensity correction. Historical feedback may also be selection-biased because outcomes are observed only for pairs selected by a prior policy. For that reason, controlled simulator rollouts with matched seeds and scenario variants are the primary comparison method.

Feedback is used only after it becomes observable. No response is not treated as a rejection, and late outcomes are not treated as failures before their observation windows mature. MSMI is sparse, so a difference driven by one or two outcomes in a small set of episodes is not conclusive.

## 6. Baselines and evaluation protocol

The policy is compared with the supplied:
- **Greedy** baseline;
- **No-clarification** baseline;
- **Random-feasible** baseline.

The evaluation covers six scenario variants: `development`, `sparse`, `cold_start`, `delayed`, `shift`, and `drift`. Runs should use the same seeds across methods and report MSMI per 100 arrived members, coverage, mutual acceptances per 100, clarification cost, waiting times, missing feedback, and inference time. Multiple seeds are necessary because the primary outcome is rare and stochastic.

### Preliminary results

The following custom-versus-greedy comparison used seeds **101, 102, and 103** across all six variants (18 episodes per method). All reported episodes were valid.

| Metric | Custom policy | Greedy baseline |
|---|---:|---:|
| Primary score: mean scenario MSMI per 100 arrived members | 0.3056 | 0.3889 |
| Coverage | 37.56% | 37.17% |
| Mutual acceptances per 100 | 6.36 | 5.50 |
| Clarification cost | 328.17 | 206.67 |
| Inference time (seconds) | 21.65 | 17.37 |

In this preliminary matched-seed comparison, the custom policy had slightly higher coverage and more mutual acceptances, but a lower primary score, higher clarification cost, and longer inference time than greedy. Thus, the current evidence does **not** support a claim that the custom policy outperforms greedy overall.

The per-scenario primary scores were:

| Scenario | Custom policy | Greedy baseline |
|---|---:|---:|
| Development | 0.3333 | 0.5000 |
| Sparse | 0.0000 | 0.1667 |
| Cold start | 0.3333 | 0.1667 |
| Delayed | 0.5000 | 0.5000 |
| Shift | 0.3333 | 0.5000 |
| Drift | 0.3333 | 0.5000 |

The custom policy performed better in the cold-start scenario in these runs, tied greedy in delayed, and performed worse in development, sparse, shift, and drift. This suggests that the clarification/scoring trade-off requires further tuning, particularly under sparse information and distribution shift.

A separate **single-seed** run (seed 101 across all six variants) produced primary scores of 0.4167 for custom, 0.5000 for greedy, 0.3333 for no-clarification, and 0.2500 for random-feasible. This is only a smoke-test comparison, not a robust estimate. No-clarification and random-feasible have not yet been run across the same three-seed set in the results reported here.

## 7. Planned ablations and next experiments

1. **Allocation:** greedy edge selection versus local two-edge improvement; consider exact maximum-weight matching if feasible within runtime constraints.
2. **Clarification:** no asks versus supplied baseline asks versus targeted asks.
3. **Scoring:** primary four-field score versus the full weighted soft-field score.
4. **Question budget:** compare hard-bundle prioritization with a candidate-edge-based value-of-information approximation.
5. **Robustness:** analyze each of the six scenario families separately and repeat with additional seeds.
6. **Efficiency:** test whether reduced clarification spend or faster scoring can preserve MSMI.

Ablations should change one component at a time and use matched seeds. The primary selection metric remains MSMI; coverage and acceptance alone are not sufficient to select a winner.

## 8. Failure cases and limitations

- Very sparse feasible graphs can limit coverage regardless of scoring quality.
- A declined field cannot be resolved by asking again or by inferring the answer.
- A myopic clarification heuristic can spend budget on information that later becomes irrelevant.
- Local two-edge improvement is not guaranteed to find a global optimum.
- Static heuristic weights may not adapt to shifted or drifting outcome patterns.
- Delayed and missing feedback make online adaptation noisy and can introduce selection bias.
- MSMI is a sparse downstream outcome; small episode counts have high uncertainty.
- The data and simulator are synthetic. These results do not establish real-world matching quality, fairness, or safety.

## 9. Reproducibility and immediate next steps

The implementation uses the provided release kit and Python standard library. The local checks completed successfully: **26 unit tests passed**, and dataset verification reported 2,000 members across 71 files. The evaluation commands completed for the preliminary runs summarized above.

Next, run the no-clarification and random-feasible baselines across the same three seeds, inspect sparse-scenario failure cases, and perform the planned ablations. Before final submission, rerun the required tests and data verification, build and test the Docker image, save machine-readable episode results, and pin the submitted source by an immutable commit SHA or archive checksum.

**Conclusion:** The constraint-first design is valid and promising as a structured starting point, but current evidence shows a trade-off rather than a win: the custom policy increases acceptance and coverage slightly while using more clarification budget and achieving a lower primary score than greedy. The next iteration should focus on improving outcome quality per unit of clarification cost, not merely increasing the number of introductions.
