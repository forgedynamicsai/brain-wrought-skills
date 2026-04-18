# ADR-003: Sealed Artifacts Architecture

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Arron Street
**Related:** ADR-001, ADR-002

---

## Context

Berkeley RDI's April 2026 paper demonstrated that all 8 major agent benchmarks are exploitable to near-perfect scores without solving tasks. Specific vulnerabilities:

- WebArena (812 tasks): navigating Chromium to `file://` URL reads gold answer directly from task config
- SWE-bench Verified (500 tasks): 10-line `conftest.py` hooks pytest and rewrites every result to "passed"
- Terminal-Bench: binary wrapper trojans on `/usr/bin/curl` produce fake passing output
- FieldWorkArena: validation pipeline never checks answer correctness

Common pattern: **scoring infrastructure trusts the submission's execution environment**. Any benchmark that ships qrels, answer keys, or judge prompts with the harness code is exploitable.

Brain-Wrought must design against this from day one or be dead on arrival.

## Decision

Brain-Wrought uses a **four-repo architecture** with strict separation of public and sealed artifacts.

### 1. Four repos

| Repo | Visibility | Contents |
|---|---|---|
| `brain-wrought-harness` | Public | pytest runner, submission CLI, config loaders, reference implementations of interfaces |
| `brain-wrought-skills` | Public | Markdown skills for fixture/query generation, **public rubric scaffolds** (not actual judge prompts), submitter docs |
| `brain-wrought-engine` | Public | Deterministic scoring code — retrieval metrics, entity recall math, F1 calculators, bootstrap CI |
| `brain-wrought-sealed` | **Private** | qrels (gold answers), actual judge rubric prompts, gold entity graphs, adversarial injection suite, held-out test fixtures |

The first three contain everything a submitter needs to understand, implement, and self-evaluate. The fourth contains everything that would trivialize the benchmark if known.

### 2. Sealed repo structure

```
brain-wrought-sealed/
├── qrels/
│   ├── retrieval_test.jsonl         # held-out test qrels for Axis A
│   ├── temporal_test.jsonl
│   ├── personalization_test.jsonl
│   └── abstention_test.jsonl
├── judge_rubrics/
│   ├── identity_private.md          # actual rubric sent to judges
│   ├── stance_private.md
│   ├── novelty_private.md
│   └── calibration_private.md
├── gold_graphs/
│   ├── fixture_001_entities.json    # ground-truth entity list per fixture
│   ├── fixture_001_backlinks.json
│   └── ...
├── adversarial/
│   ├── prompt_injection_suite.md    # tests for judge robustness
│   └── scoring_exploit_tests.md     # tests that submissions aren't exploiting
└── test_fixtures/
    ├── clean_held_out/              # fixture vaults never released
    └── dirty_held_out/
```

### 3. Access controls

- **Write access:** Arron only (enforced via GitHub branch protection + 2FA)
- **Read access:** Arron + GitHub Actions CI (service account with minimal token)
- **Audit log:** Enable repo access logs; review monthly
- **Forking:** Disabled on private repo
- **Cloning outside CI:** Technically possible for Arron; should avoid to prevent accidental exposure

### 4. CI-time access pattern

Official eval flow:
1. Submitter pushes Docker image to GHCR
2. GitHub Actions workflow in `brain-wrought-harness` triggers
3. Workflow authenticates to `brain-wrought-sealed` via GitHub Actions OIDC (no long-lived token)
4. Fetches specific commit SHA of sealed content (pinned in workflow config)
5. Mounts sealed content read-only into harness container (NOT submission container)
6. Harness runs submission container with `--network none`
7. Harness collects submission outputs, scores against sealed qrels
8. Sealed content discarded when workflow ends
9. Only final scores are persisted

Critically: **submission container never sees sealed artifacts.** Sealed content lives in the harness container; harness invokes submission container with prompts/queries from sealed content but submission receives only the prompt, not the qrel or rubric.

### 5. Public-private boundary

**Public (submitters can see):**
- Harness code, submission interface, CLI
- Engine code (scoring math, entity extraction logic)
- Public dev qrels (~100 retrieval queries, 30 ingestion tasks, 10 assistant tasks) — subset of test set, useful for self-eval
- Public fixture samples (5 clean brains, 5 dirty brains) — for development reference
- Public rubric scaffolds — dimensions + example responses, NOT actual prompts
- Entity schemas, backlink conventions, frontmatter formats

**Sealed (never released):**
- Full test qrels (~500 queries across all axes)
- Actual judge rubric prompts
- Complete gold entity graphs for all test fixtures
- Held-out test fixtures (~100 clean + 100 dirty + 30 adversarial)
- Adversarial injection suite

### 6. Rotation schedule

- Public fixtures: frozen (v1 samples stay v1 samples)
- Public dev qrels: can expand over time; scores under old vs new clearly marked
- Sealed qrels: refreshed every 6 months; old qrels archived, new qrels used for new submissions
- Sealed rubrics: rotated every 90 days (see ADR-002)
- Adversarial injection suite: rotated every 30 days (hot target)

### 7. What to do if sealed content leaks

Contingency plan:
1. Immediately rotate all leaked content
2. Flag all submissions since the likely leak date for audit
3. Publicly disclose the incident in a blog post and Arxiv update
4. Post-mortem documented in `/SECURITY.md`

Leak detection:
- Monitor submission scores for sudden jumps (possible overfitting to leaked qrels)
- Monitor for submissions that over-perform on test set relative to dev set (anomaly)
- Watch GitHub for accidentally-forked content

## Alternatives considered

### Single public repo with obfuscated qrels

**Rejected.** "Obfuscated" = reversible with enough effort. Berkeley's conftest.py exploit took ~10 lines. Obfuscation is not a security strategy.

### Encrypted qrels in public repo

**Rejected.** Key management becomes the weak link. If the key is in CI, it's recoverable. If it rotates, old submissions can't be re-scored. Not worth the complexity.

### Hosted qrels API (submissions query during eval)

**Rejected for v1.** Would allow timing analysis (measure API latency to infer content), introduce service-availability risks, and require hosted infrastructure Arron doesn't want to run. Revisit for v2 if scale demands.

### Qrel generation on-the-fly (no stored qrels)

**Rejected.** Would require deterministic-seeded LLM generation at eval time, which undermines reproducibility. Also expensive.

### Public everything (trust submitters)

**Rejected.** Berkeley paper is the counter-argument. Trusting submitters is what makes benchmarks gameable.

## Consequences

### Positive

- Berkeley exploit patterns don't apply — sealed content never touches submission containers
- Clear legal/intellectual-property boundary (public MIT licensed, sealed is Arron's work product)
- CI-based access means no long-lived secrets to manage
- Rotation schedule keeps the benchmark fresh even if partial leaks occur

### Negative

- Submitters can't fully debug their systems without official eval (mitigated by generous public dev set)
- "Sealed" reduces transparency, which some academic reviewers will criticize (mitigated by sealed qrel audit — see REPRODUCIBILITY_SPEC §6)
- Single-maintainer model for sealed content is a bus factor risk
- Community can't contribute to sealed content directly (workaround: curation via public proposals)

### Neutral

- License split: public repos MIT, sealed repo all-rights-reserved (Arron retains copyright, grants Brain-Wrought exclusive license). Documented in each repo's LICENSE file.

## Review conditions

Revisit this ADR if:
- Scale exceeds single-maintainer capacity → add trusted maintainer(s)
- Academic reviewer consistently flags sealed-as-problematic → consider limited public audit-subset
- Any leak occurs → rotate + reassess controls
- Berkeley or NeurIPS publishes updated guidance on benchmark design → align
