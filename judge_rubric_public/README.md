# Public Judge Rubric Scaffolds

These files describe the **dimensions** being evaluated by Axis C judges, with examples of strong and weak responses. They do NOT contain the actual judge prompts used at eval time (those are sealed).

Per ADR-002, sealed rubrics are rotated quarterly. The public scaffolds here are stable references for submitters to understand what judges look for.

## Dimensions

### Identity Accuracy
Does the response correctly reflect who the user is? Strong responses draw on documented facts from the brain (role, context, history). Weak responses make up facts or confuse the user with other entities.

### Stance Coherence
Does the response represent the user's documented views? Strong responses reflect stated opinions and preferences from existing notes. Weak responses contradict documented stances or invent positions.

### Novelty vs. Regurgitation
Is the system synthesizing or just paraphrasing notes? Strong responses connect information across notes, draw non-obvious conclusions, surface patterns. Weak responses quote notes verbatim without synthesis.

### Calibration
Did the system abstain when it should have? Strong responses: confident when evidence is strong, hedged when uncertain, abstained when the brain genuinely doesn't contain an answer. Weak responses: confidently wrong, or over-abstaining when evidence is present.

## Scoring

Each dimension is scored 1-5 by a panel of 3 judges (Sonnet 4.6, Opus 4.7, GPT-5.4). Panel score = median. Results reported as mean ± 95% bootstrap CI across all tasks.

See [ADR-002](../adr/ADR-002-judge-panel.md) for full judge panel design rationale.
