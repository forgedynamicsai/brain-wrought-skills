# ADR-001: Three-Axis Framework

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Arron Street
**Supersedes:** N/A

---

## Context

Brain-Wrought evaluates AI systems that read, maintain, and reason over personal knowledge vaults. The question is what dimensions to evaluate on.

Existing benchmarks handle narrow slices:
- RAGAS: RAG quality (answer faithfulness, context relevance, answer relevance)
- MTEB: embedding quality on retrieval tasks
- HELMET: long-context comprehension
- LongMemEval: conversational memory
- GAIA: general assistant (exploited by Berkeley)
- PrivacyBench: privacy leakage in personalized assistants
- PersonaBench / LaMP: persona consistency
- gbrain eval: retrieval-only (P@k, Recall@k, MRR, nDCG@k) for personal brains

None evaluates the full personal-brain loop: ingest raw inputs → organize into structured knowledge → retrieve from it → personalize outputs.

Without a framework that covers the full loop, vendors optimize for narrow metrics and the system-as-a-whole remains unmeasured.

## Decision

Brain-Wrought evaluates three axes:

### Axis A — Retrieval

**Question:** Does the brain find what's already filed?

**Metrics:**
- P@k, Recall@k, MRR, nDCG@k (extending gbrain eval pattern)
- Personalization P@k (whose-note-is-this weighting — does the system surface the right user's note when brains are merged?)
- Temporal qrels ("who did I meet last Tuesday?", "what did I decide about X in March?")
- Abstention rate (% of adversarial queries correctly declined)

**Fixture tiers:**
- Clean schema: perfect frontmatter, timelines, backlinks, consistent tags
- Dirty schema: stubs, missing fields, stale dates, broken links — a real brain in the wild

A great brain scores well on both.

### Axis B — Ingestion

**Question:** Can the agent turn raw inbox content into a well-structured brain?

Given synthetic inbox: emails, PDFs/xlsx/docs in a project folder, calendar dump, Slack/iMessage simulation. The agent must write brain pages with proper structure.

**Metrics:**
- Entity recall (% of ground-truth entities extracted and given pages)
- Backlink F1 (vs. curated gold entity graph)
- Citation accuracy (do quoted passages match sources?)
- Schema completeness rubric (frontmatter fields, exec-summary, state, see-also, timeline — the pattern from Garry Tan's `~/git/brain`)
- Setup friction (commands + prompts the human had to provide to make it useful)

This is the axis nobody measures today, and the one that most differentiates a real brain from a sophisticated search engine.

### Axis C — Assistant / Personalization

**Question:** Does the agent actually know the user?

Tasks:
- Proactive morning brief (what's on my calendar, who am I meeting, what's changed since yesterday)
- Meeting prep (given an upcoming meeting, what should I know?)
- Next-best-action (what am I forgetting?)
- Synthesis ("what does the brain think about X?")

Scored by a sealed LLM-judge panel on:
- Identity accuracy (does the response correctly reflect who the user is?)
- Stance coherence (does it represent the user's documented views?)
- Novelty vs. regurgitation (is the system synthesizing or just paraphrasing notes?)
- Calibration (did it abstain when it should have?)

Fuzziest axis. Reported with bootstrap confidence intervals.

## Composite score

```
composite = 0.35 * retrieval + 0.35 * ingestion + 0.30 * assistant
```

Weighting rationale:
- Retrieval and ingestion are more objective; heavier weight.
- Assistant is LLM-judged; lighter weight to reduce noise dominance.
- Equal retrieval and ingestion because both are fundamental — without ingestion there's no brain to retrieve from.

Individual axis scores also published; composite is for leaderboard ranking only.

## Alternatives considered

### Single-axis benchmark (retrieval only)

**Rejected.** Fastest to ship but doesn't differentiate from gbrain eval. Real personal-brain systems live or die on the full loop; measuring one third misses the point.

### Four-axis framework (adding "maintenance")

**Rejected for v1.** Considered adding a fourth axis — does the brain maintain itself over time (fix citations, consolidate memory, deduplicate)? This is real and important, but:
- Requires longitudinal evaluation (weeks of wall-clock)
- Current agent systems don't consistently implement maintenance
- Would push v1 out 2-3 months
Deferred to v1.5 or v2.

### Five-axis framework (adding "privacy")

**Rejected for v1.** PrivacyBench exists and does this well. Brain-Wrought can cite and integrate PrivacyBench as an optional submodule in v1.1.

### Task-based evaluation only (no axis decomposition)

**Rejected.** Aggregate task scores obscure what a system is good at. Axis decomposition lets a retrieval-only system submit and score honestly on what it implements.

## Consequences

### Positive

- Clear, named axes make submission comparison intelligible to non-experts
- Partial submissions are valid (retrieval-only is still a real contribution)
- Each axis can ship publicly on its own schedule (enables warpspeed phasing)
- Gold-standard alignment: the axes map cleanly to RAGAS, MTEB, LongMemEval, PersonaBench territory — we're not inventing new metrics where good ones exist

### Negative

- Weighting choice is contested. Someone will argue ingestion should be 0.45 or assistant should be 0.40. Accept and document; allow sub-community leaderboards to reweight.
- Axis independence assumption is imperfect. A great ingestion system indirectly helps retrieval. Mitigated by using separate fixtures for each axis evaluation.
- LLM-judged axis introduces variance. Mitigated by panel voting + bootstrap CI + rubric red-teaming (see ADR-002).

## Review conditions

Revisit this ADR if:
- v1 launches and community feedback converges on different weightings
- A new axis becomes clearly essential (e.g., maintenance in v2)
- PrivacyBench integration forces a structural change
