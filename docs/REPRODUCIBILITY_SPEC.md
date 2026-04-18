# REPRODUCIBILITY_SPEC

Brain-Wrought's central claim is that a third party can rerun any submission and get the same scores within stated confidence intervals.

---

## 1. Principles

1. **Everything deterministic that can be deterministic.** Retrieval metrics, entity extraction, F1 calculations — no LLM in the loop.
2. **Seed everything that isn't.** Fixture selection, randomization, judge order, bootstrap sampling — all seeded from submission hash.
3. **Pin everything.** Model versions, library versions, Docker base images, Python interpreter.
4. **Prove it continuously.** Public CI runs the full reproduction on every commit to main.
5. **Make it a one-liner for submitters.** `docker run brain-wrought/eval:v1.0 --submission <path>` produces a JSON result.

---

## 2. Determinism boundaries

### 2.1 Fully deterministic (must be bit-identical)

- Retrieval scoring: P@k, Recall@k, MRR, nDCG@k
- Entity recall (given extracted entity list)
- Backlink F1 (given extracted graph)
- Citation accuracy
- Schema completeness rubric
- Composite score aggregation

### 2.2 Seeded-stochastic (must match given seed)

- Fixture selection from pool (seed = submission_hash)
- Query order randomization (seed = submission_hash)
- Bootstrap confidence intervals (seed = submission_hash)

### 2.3 Bounded-stochastic (must fall within declared CI)

- Judge panel scoring (three LLM judges, non-zero temperature)
- Entity extraction from unstructured inbox content

---

## 3. Pinning

### 3.1 Model versions

```yaml
judges:
  - provider: anthropic
    model: claude-sonnet-4-6
    temperature: 0.1
    max_tokens: 1024
  - provider: anthropic
    model: claude-opus-4-7
    temperature: 0.1
    max_tokens: 1024
  - provider: openai
    model: gpt-5.4-2026-XX-XX  # pin snapshot ID when available
    temperature: 0.1
    max_tokens: 1024
```

### 3.2 Software versions

`pyproject.toml` pins exact versions. No `latest`, no `*`, no `^`. Lock file committed.

### 3.3 Docker base image

```dockerfile
FROM python:3.12.3-slim-bookworm@sha256:<digest>
```

SHA digest, not just tag.

---

## 4. Seed propagation

```python
import hashlib

def submission_seed(docker_digest: str, harness_version: str) -> int:
    h = hashlib.sha256(f"{docker_digest}:{harness_version}".encode()).hexdigest()
    return int(h[:16], 16)

seed = submission_seed(...)
fixture_rng = Random(seed)
query_rng = Random(seed + 1)
judge_order_rng = Random(seed + 2)
bootstrap_rng = Random(seed + 3)
```

---

## 5. Docker container

Official eval only runs inside `brain-wrought/eval:v{version}`. Image published to GHCR, signed with cosign, SBOM attached via syft.

**Self-eval mode** (public dev set only):

```bash
docker run --rm -v $(pwd)/submission:/submission \
  brain-wrought/eval:v1.0 \
  --submission /submission \
  --mode self-eval
```

**Official eval mode** (CI only):

```bash
docker run --rm \
  -e BRAIN_WROUGHT_CI_TOKEN=$SEALED_REPO_TOKEN \
  -v $(pwd)/submission:/submission \
  brain-wrought/eval:v1.0 \
  --submission /submission \
  --mode official \
  --seed-from-hash
```

---

## 6. Submitter requirements

1. Docker image (no naked Python scripts)
2. Harness stdin/stdout interface
3. All dependencies pinned
4. Model temperatures and versions declared
5. `--network none` compliant
6. Completes within 60-minute wall-clock limit

---

## 7. Audit process

Top-5 submissions: pull image, inspect Dockerfile, run on a fresh machine, verify scores reproduce. Publish audit report. Failing audits are delisted.

---

## 8. Reproducibility limits

- Not bit-identical across hardware architectures (float x86 vs ARM)
- Not identical across silent model provider updates
- Not immune to time-based drift (mitigated by synthetic fixtures, no live web data)
