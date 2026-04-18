# ADR-002: Judge Panel + Rubric Sealing

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Arron Street
**Related:** ADR-001, ADR-003

---

## Context

Axis C (assistant/personalization) cannot be scored by deterministic code — it requires judging the quality and correctness of an LLM-generated response. The standard pattern is LLM-as-judge, but this has known failure modes:

- **Single judge bias:** one model's preferences dominate scoring
- **Prompt injection:** submissions inject instructions that trick the judge (Berkeley RDI, Apr 2026)
- **Rubric leakage:** if the judge prompt is public, submissions over-fit to it
- **Non-determinism:** same response, different scores on reruns

Brain-Wrought needs a judge design that handles all four.

## Decision

### 1. Three-judge panel with majority vote

Panel composition:
- **Judge A:** Anthropic Claude Sonnet 4.6 (temperature 0.1)
- **Judge B:** Anthropic Claude Opus 4.7 (temperature 0.1)
- **Judge C:** OpenAI GPT-5.4 (temperature 0.1)

Same-vendor concentration is 67% (A+B are both Anthropic). Mitigation: pair Anthropic judges with one non-Anthropic judge as cross-check. Next-best non-Anthropic candidate is Google Gemini 3.1 Pro — keep as a fallback if GPT-5.4 access becomes constrained.

**Voting rule:**
- Each judge produces scores on four rubrics: identity accuracy, stance coherence, novelty vs. regurgitation, calibration
- Each rubric scored 1-5
- Panel score per rubric = median of three judges (robust to single-judge outlier)
- If standard deviation across judges > 1.5 on any rubric: flag the sample as "contested," review manually before inclusion in score

### 2. Rubric sealing

Four rubrics, each a sealed Markdown file in `brain-wrought-sealed/judge_rubrics/`:
- `identity_private.md` — Does the response correctly reflect who the user is?
- `stance_private.md` — Does the response represent the user's documented views?
- `novelty_private.md` — Is the system synthesizing or just paraphrasing notes?
- `calibration_private.md` — Did the system abstain when it should have?

**Why sealed:** Berkeley's April 2026 paper showed submissions can game public rubrics by over-fitting. Sealed rubrics force submissions to solve the underlying quality problem, not the specific rubric wording.

**Public rubric scaffolds** in `brain-wrought-skills/judge_rubric_public/` — describe the dimensions being measured and give examples of strong/weak responses, but don't contain the exact prompts judges receive.

### 3. Rubric loading at eval time

At official eval time, the harness:
1. Authenticates to `brain-wrought-sealed` via CI token (GitHub Actions OIDC)
2. Fetches rubrics from pinned commit SHA
3. Passes rubrics to judges as system prompts
4. Sealed content never persists in submission containers

### 4. Rubric rotation

Every 90 days:
- New rubric versions written by project maintainer
- Red-teamed against: (a) the adversarial injection suite, (b) top-5 prior submissions to check for regression
- Released as new `qrel_version`; old submissions retain scores under old version
- Published in leaderboard as footnote ("scored under rubric version 2026-Q2")

### 5. Bootstrap confidence intervals

For each axis-C task:
- Panel scores sample: 3 judges × 1 rubric × 1 task = 3 data points
- Across N tasks (50 for v1): bootstrap 1000 resamples
- Report mean ± 95% CI

This makes small score differences appropriately uncertain. Leaderboard displays CI alongside point estimate.

### 6. Judge prompt engineering standard

All rubric prompts must pass Arron's 10-principle prompt framework:
1. Sharply defined role (evaluator with domain expertise)
2. Unambiguous imperatives (must/shall/never)
3. Strict output format (JSON with fixed keys)
4. Non-negotiable guardrails (no scoring above 3 without citation to evidence)
5. Token efficiency (rubric < 1500 tokens; cache aggressively)
6. Few-shot exemplars (3 strong + 3 weak responses, per rubric)
7. Explicit CoT ("Before scoring, list 3 specific evidence points")
8. Self-evaluation ("Verify your score is consistent with the evidence you listed")
9. Precise domain terminology (no "quality" — score specific dimensions)
10. Mission-critical instructions at beginning AND end (primacy + recency)
11. Versioning (`rubric_version: 2026-Q2` in every output)

## Alternatives considered

### Single judge (Opus 4.7 only)

**Rejected.** Deterministic within run but no cross-validation, high susceptibility to prompt injection, no way to detect biased judging.

### Human judges

**Rejected for v1.** Gold standard but:
- Doesn't scale to 500+ submissions/year
- Introduces its own biases (IRR challenges, cultural variation)
- Cost-prohibitive
Considered for post-hoc audit of top-5 submissions (see REPRODUCIBILITY_SPEC.md §8).

### Five-judge panel

**Rejected for v1.** Better statistical power but 67% more expensive per submission (5 vs 3). Diminishing returns past 3. Revisit if inter-rater agreement (Fleiss kappa) on pilot < 0.4.

### Public rubrics

**Rejected.** Berkeley paper showed why. Public rubric scaffolds (not the actual judge prompts) serve the educational purpose for submitters.

### Judge model ensemble with same family

**Rejected.** Sonnet-only or Opus-only panel has correlated failure modes. Cross-vendor (Anthropic + OpenAI, or Anthropic + Google) forces actual diversity of judgment.

## Consequences

### Positive

- Three-judge majority + bootstrap CI addresses single-judge failure modes
- Sealed rubrics + rotation resist over-fitting and Berkeley-style exploits
- 10-principle compliance makes rubrics defensible when audited
- LiteLLM-based router means adding/swapping judges is a config change

### Negative

- Judge cost dominates runtime cost (~$2.50 of $3.30 per submission)
- Two Anthropic judges creates correlation risk; partially mitigated by GPT-5.4 as third and Gemini as fallback
- Sealed rubrics raise fair-evaluation concerns; mitigated by published scaffolds, audit process, and appeal rights (SUBMISSION_PROTOCOL.md §10)
- Rubric rotation adds maintenance overhead (quarterly cadence)

## Review conditions

Revisit this ADR if:
- Inter-rater agreement (Fleiss kappa) on pilot < 0.4 → consider 5-judge or human pilot
- Any judge deprecates → swap; re-baseline all submissions
- Cost per submission > $10 → reduce panel or task count
- Community reports systematic bias → audit rubrics and rotate earlier
