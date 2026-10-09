# Round 1 Research Note — The Sequential Matching Problem

**Approach:** Constraint-first sequential allocation with targeted clarification  
**Data:** Public synthetic release 1.0.0  
**Status:** Proposed method and initial implementation plan; no private-world results are claimed.

## 1. Problem interpretation

The policy makes two decisions each simulated day: which unanswered fields to clarify under a limited budget, and which available pairs to introduce. The policy only sees arrived members, currently observable profile fields, field-status indicators, past introductions, feedback whose observation time has arrived, and policy-owned memory. It must not infer or use future arrivals, hidden simulator traits, private response propensities, or unobserved future outcomes.

An introduction is valid only when reciprocal hard constraints are satisfied, both people are available, the pair has not previously been introduced, and neither person is used twice in the same batch. Missing hard information is not treated as approval. A known hard-constraint failure makes a pair infeasible; unresolved hard information requires clarification.

The primary objective is mutual second-meeting intention (MSMI) per 100 arrived members. Coverage, mutual acceptance, clarification cost, and inference time are relevant secondary metrics/tie-breakers. Missing or delayed feedback must not be treated automatically as a negative outcome.

## 2. Hypothesis

A policy that (1) applies the official reciprocal feasibility function, (2) spends clarification budget on members whose missing hard constraints affect many plausible candidate edges, and (3) allocates non-overlapping introductions using compatibility-weighted edges with local allocation improvements will outperform simple greedy matching on at least some scenarios, while strict feasibility and budget checks preserve episode validity. Benefits are expected to be most relevant when profiles are incomplete; performance under sparse, delayed, shifted, and drifting conditions must be measured rather than assumed.

## 3. Candidate allocation method

1. Consider only currently available members and exclude previously introduced pairs.
2. Call the release's `eligibility(a, b)` function; retain only `feasible` edges. Never override an infeasible or unresolved hard constraint with a soft score.
3. Score edges using observable soft-field agreements. The initial heuristic emphasizes `relationship_goal`, `relationship_pace`, `lifestyle`, and `conversations`; other soft fields receive smaller tie-breaking weights. Unknown values contribute no positive or negative evidence and are not imputed as mismatches.
4. Seed a non-overlapping batch by descending edge utility with stable tie-breaking.
5. Attempt local two-edge replacements on the same four members when they increase total edge utility. This is a deterministic local-improvement heuristic, not an exact maximum-weight matching solver; a later ablation should determine whether a stronger optimizer is worth its complexity.

The weights are initial engineering hypotheses, not fitted parameters and not real-world relationship estimates. The simulator's outcome mechanism is synthetic and scenario-dependent, so the weights must be tested across scenario families.

## 4. Clarification policy

The daily clarification budget is 12 units. A `constraints` bundle costs 3 units and named soft-field asks cost 1. The first implementation ranks available members with unresolved, non-declined hard fields by the number of candidate edges currently marked `needs_clarification`, with a small preference for members who also have feasible alternatives. It asks up to four high-priority hard bundles first. If budget remains, it asks primary soft fields for members with feasible alternatives, prioritizing members with more such alternatives.

Declined answers remain unknown and are never silently inferred. The policy does not repeatedly ask a hard bundle when a hard field has already been declined. This heuristic is intentionally simple; a more formal value-of-information score can be evaluated after a valid baseline is established.

## 5. Missingness and delayed feedback

`null` means unknown, not zero or false. `field_status` distinguishes observed, not asked, and declined. The policy makes decisions only from the current request. Historical introduction records with null propensity do not support inverse-propensity correction, and observed historical outcomes may be selection-biased. Therefore, the primary comparison should use controlled simulator rollouts with the same seeds and scenario variants, rather than treating historical feedback as an unbiased offline test set.

Feedback is used only after it becomes observable. A no-response event is not interpreted as a rejection. MSMI is sparse and depends on downstream events, so small differences in a small number of episodes should not be treated as conclusive.

## 6. Evaluation plan and baselines

Compare the custom policy against the supplied:

- greedy baseline;
- no-clarification baseline;
- random-feasible baseline.

Use identical public evaluation seeds and all six scenario families: `development`, `sparse`, `cold_start`, `delayed`, `shift`, and `drift`. Report per-scenario MSMI per 100 arrived members, coverage, mutual acceptances per 100, clarification cost, waiting-time distribution, missing feedback, and inference time. Repeat across multiple seeds before making claims.

## 7. Planned ablations

1. Greedy edge allocation versus the local-improvement allocation.
2. No asks versus supplied baseline asks versus targeted asks.
3. Primary four-field score versus the full weighted soft-field score.
4. Remove the local two-edge improvement step.
5. Compare performance separately across sparse, cold-start, delayed, shift, and drift scenarios.

## 8. Failure cases and limitations

- If few pairs are feasible, coverage can remain low even with good allocation.
- Clarification may not resolve fields explicitly declined by a member.
- A myopic question policy can spend budget on edges that become irrelevant after other answers arrive.
- Greedy initialization plus local two-edge changes is not guaranteed to find the global optimum.
- Static heuristic weights may fail when scenario weights shift or drift.
- Delayed and missing feedback makes online learning noisy and potentially biased.
- The public data and simulator are synthetic; results would not establish real-world matching quality or safety.

## 9. Reproducibility and next steps

The implementation uses Python's standard library and the provided release kit. First validate the policy protocol and constraints, then run a single development seed as a smoke test, then run matched-seed baseline comparisons and ablations. Save machine-readable results and report scenario-level results; do not report unrun experiments as findings. Before final submission, run the supplied unit tests, data verification, and Docker evaluation, and pin the submitted source by an immutable commit or archive checksum.
