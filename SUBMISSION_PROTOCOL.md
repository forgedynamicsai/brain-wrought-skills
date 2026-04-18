# SUBMISSION_PROTOCOL

This document specifies what external teams must implement to submit a personal brain system to Brain-Wrought for evaluation. It's the public contract.

---

## 1. Overview

A Brain-Wrought submission is a Docker container that:
1. Accepts a personal brain vault at `/brain` (mounted read/write)
2. Implements a standard CLI interface for ingestion, retrieval, and assistant modes
3. Runs offline (no network access)
4. Produces JSON output on stdout
5. Completes within a wall-clock time limit

Submissions are evaluated across three axes. A submission may decline to implement one or more axes; it will score zero on unimplemented axes.

---

## 2. Container requirements

### 2.1 Dockerfile

```dockerfile
FROM python:3.12.3-slim-bookworm  # or any pinned base

COPY requirements.txt /tmp/
RUN pip install --no-cache-dir -r /tmp/requirements.txt

COPY . /app
WORKDIR /app

ENTRYPOINT ["python", "submission_entry.py"]
```

### 2.2 No network access

The harness runs your container with `--network none`. Everything your system needs — including LLM inference — must either be packaged inside the container (local models) or called through the harness's provided LLM proxy (see section 5).

### 2.3 Resource limits

- CPU: 8 cores
- RAM: 16GB
- Disk: 50GB writable at `/brain`
- GPU: optional; declare `gpu: true` in `submission.yaml`

---

## 3. Entrypoint interface

Your submission receives commands as stdin JSON and emits results as stdout JSON.

### 3.1 Ingestion mode

```json
// stdin
{
  "mode": "ingest",
  "inbox_path": "/inbox",
  "brain_path": "/brain",
  "seed": 1234567890
}
```

```json
// stdout
{
  "status": "ok",
  "entities_extracted": 47,
  "backlinks_created": 156,
  "notes_written": 89,
  "setup_prompts_required": 0,
  "elapsed_seconds": 423
}
```

### 3.2 Retrieval mode

```json
// stdin
{
  "mode": "retrieve",
  "brain_path": "/brain",
  "query": "who did I meet last Tuesday?",
  "query_type": "temporal",
  "k": 10,
  "seed": 1234567890
}
```

```json
// stdout
{
  "results": [
    {"note_id": "people/alice.md", "score": 0.95, "excerpt": "..."}
  ],
  "abstained": false,
  "abstention_reason": null,
  "elapsed_ms": 142
}
```

### 3.3 Assistant mode

```json
// stdin
{
  "mode": "assist",
  "brain_path": "/brain",
  "task": "Give me my morning brief for 2026-05-15",
  "task_type": "morning_brief",
  "context": {},
  "seed": 1234567890
}
```

```json
// stdout
{
  "response": "...",
  "citations": [
    {"note_id": "projects/alpha.md", "quote": "..."}
  ],
  "confidence": 0.78,
  "elapsed_ms": 3421
}
```

Citations are required. Uncited claims are treated as fabrications by the judge panel.

---

## 4. submission.yaml

```yaml
name: "my-personal-brain-v1"
team: "alice@example.com"
github: "alice/my-personal-brain"
paper_url: "https://arxiv.org/abs/..."  # optional

axes:
  retrieval: true
  ingestion: true
  assistant: true

models:
  - role: "retrieval"
    provider: "local"
    identifier: "sentence-transformers/all-mpnet-base-v2"
    version: "2.2.2"

resources:
  gpu: false
  ram_gb: 8

license: "MIT"
```

---

## 5. LLM proxy (optional)

If your submission uses commercial LLM APIs, route through the harness's LLM proxy at `http://brain-wrought-proxy:8080` inside the container. Usage covered by Brain-Wrought up to $5 per submission.

---

## 6. Time limits

| Mode | Per-call limit | Total wall-clock |
|---|---|---|
| Ingestion | 20 minutes | 20 minutes |
| Retrieval | 10 seconds | 15 minutes (100 queries) |
| Assistant | 30 seconds | 25 minutes (50 tasks) |
| **Total** | | **60 minutes** |

---

## 7. Self-evaluation

```bash
git clone https://github.com/forgedynamicsai/brain-wrought-harness
cd brain-wrought-harness

docker build -t my-brain:v1 /path/to/your/submission

brain-wrought self-eval --submission my-brain:v1
```

---

## 8. Official submission

```bash
brain-wrought submit \
  --image my-brain:v1 \
  --submission-yaml ./submission.yaml \
  --contact alice@example.com
```

---

## 9. Audit requirements

Top-5 leaderboard submissions undergo manual audit. Dockerfile reviewed, fresh reproduction run, audit report published alongside leaderboard entry. Failed audits are delisted.

---

## 10. Appeals

Open a GitHub issue on `brain-wrought-harness` with label `appeal` within 14 days. Maintainer responds within 7 days.

---

## 11. What's NOT allowed

- Network access during eval (enforced by `--network none`)
- Reading sealed artifacts from the harness container
- Prompt injection targeting the judge panel
- Sockpuppet submissions

Violators are publicly delisted and banned for 12 months.
