# ADR-004: Reproducibility as First-Class Requirement

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Arron Street
**Related:** ADR-001, ADR-002, ADR-003

---

## Context

The broader claim of Brain-Wrought — "a credible neutral benchmark" — is unsupportable without reproducibility. Academic venues (NeurIPS 2027 ED Track) and community adoption both require that any reviewer or external team can clone the repo, run the eval, and get identical scores. The Berkeley RDI April 2026 exploit paper (which motivated this project) demonstrated that unreproducible benchmarks are trivially gamed; reproducibility is therefore also a Berkeley-proof design requirement.

The temptation in a warpspeed build is to defer reproducibility infrastructure to v1.1. This is wrong:
- Retrofitting pinned dependencies is painful (every library has drifted by then)
- Retrofitting deterministic seeds requires auditing every stochastic operation
- Retrofitting Docker is possible but adds weeks
- Retrofitting public CI means changing how every commit gets tested

Cheaper to build in from day one.

## Decision

Reproducibility is a first-class, non-negotiable requirement for every phase. Full spec in `REPRODUCIBILITY_SPEC.md`. Summary:

### 1. Determinism boundaries (documented per operation)

Each scoring operation is classified as:
- **Fully deterministic** — must produce bit-identical output (with IEEE 754 float caveat)
- **Seeded-stochastic** — must produce identical output given the same seed
- **Bounded-stochastic** — reruns must fall within declared confidence interval

Every operation in `brain-wrought-engine/` is annotated with its determinism class in docstrings and tested against that claim in CI.

### 2. Pin everything

- Python: exact version (`python = "3.12.3"`)
- Libraries: exact versions with lock file committed (`poetry.lock` or `uv.lock`)
- Docker base: SHA256 digest, not just tag
- Model versions: snapshot IDs where providers expose them (e.g., `claude-opus-4-7`, `gpt-5.4-2026-XX-XX`)
- Judge temperatures: declared (0.1 for all judges)
- Seeds: derived from submission_hash, documented

### 3. Deterministic seed propagation

Every random operation derives its seed from `submission_hash` (SHA256 of Docker image digest + harness version). Enforcement:
- Import `random.Random()` not `random.<anything>` (no global state)
- Import `numpy.random.Generator` not `numpy.random.<anything>`
- Every call documented with seed source

Audit via lint rule in CI: any use of global random state → build fails.

### 4. Dockerized eval harness

Official eval only runs inside `brain-wrought/eval:v{version}` Docker image. Submission Dockerfile must follow the pattern in `SUBMISSION_PROTOCOL.md`. Images:
- Published to GHCR with SHA digest in release notes
- Signed with cosign (keyless via GitHub OIDC)
- SBOM attached to each release (generated via syft)

### 5. Public CI as living proof

GitHub Actions workflow on every main-branch commit:
1. Build Docker image from scratch (no cache)
2. Verify image digest matches declared
3. Run eval on all 3 reference submissions
4. Compare scores to committed `reference_scores.json`:
   - Deterministic scores: must match exactly
   - Stochastic scores: must fall within declared CI
5. Build fails if any check fails

This is the public attestation that reproducibility holds. It also catches silent model-provider updates that would otherwise drift scores.

### 6. Submitter reproducibility requirements

Documented in `SUBMISSION_PROTOCOL.md`:
- Must be a Docker image (not a naked Python script)
- Must pin all dependencies
- Must declare model temperatures and versions
- Must run with `--network none` (no hidden external calls)
- Must complete within 60-minute wall-clock limit

Submissions that can't reproduce their own scores in manual audit (top-5) are delisted.

### 7. Versioning discipline

- **Semver on the harness.** Major version bump = breaking change to submission interface. Minor = new features, backward-compatible. Patch = bug fixes, same scores expected.
- **Calendar versioning on qrels.** `qrel_version: 2026-Q2`. Leaderboard clearly marks which submissions ran under which qrel version.
- **Snapshot judge pinning.** Each release declares exact judge model IDs. Model deprecation → new release with migration notes.
- **Harness version stamped in every submission record.** No ambiguity about what produced a score.

## Alternatives considered

### Reproducibility by social contract (no enforcement)

**Rejected.** Works until someone cheats; then the benchmark collapses. Berkeley paper shows the realistic attack surface.

### Full bit-level determinism (even on stochastic axes)

**Rejected.** Impossible with LLM judges at non-zero temperature. Bounded-stochastic with bootstrap CI is the honest compromise.

### Defer reproducibility to v1.1

**Rejected.** Retrofit cost > build-in cost. Also, AgentX submission without reproducibility is a weak submission.

### Rely on submitters to self-report reproducibility

**Rejected.** Self-reports are noise. Manual audit of top-5 is the enforcement teeth.

## Consequences

### Positive

- AgentX submission is stronger on day one
- Submissions that can't reproduce are delisted, preserving leaderboard integrity
- Public CI as living proof is itself a credibility artifact — anyone can verify
- Future Arron doesn't pay the retrofit cost

### Negative

- Docker requirement may exclude some informal submitters who'd prefer naked Python scripts. Mitigated by providing a template Dockerfile.
- Pinning model snapshots creates a migration cost when providers deprecate. Acceptable; document migration in release notes.
- Manual audit of top-5 is labor Arron must do. Acceptable at expected scale (~5-20 top submissions/year).
- CI runtime cost: full eval on 3 reference submissions per commit = ~$10/commit. Manageable with commit-path filtering (only run full eval on main-branch merges).

### Neutral

- The Docker requirement mirrors SWE-bench's container-based eval; submitters familiar with that ecosystem should find the pattern natural.

## Review conditions

Revisit this ADR if:
- Reproduction failures (same seed, different scores) exceed 1% across audit samples → investigate root cause
- Model provider deprecation cadence exceeds quarterly → invest in local-model fallbacks
- CI cost exceeds $100/month → add filtering (e.g., full eval only on release tags)
- Community reports the Docker barrier is too high → provide optional cloud-hosted eval service in v2
