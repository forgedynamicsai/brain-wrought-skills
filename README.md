# brain-wrought-skills

Markdown skills, rubric scaffolds, and submitter documentation for Brain-Wrought.

## What this is

- **Fixture generation skills** — templates for generating clean-schema and dirty-schema synthetic brains
- **Query generation skills** — templates for temporal, personalization, factual, and abstention qrels
- **Public rubric scaffolds** — dimensions and example responses for Axis C judging (NOT the actual sealed judge prompts)
- **Submitter docs** — SUBMISSION_PROTOCOL.md, REPRODUCIBILITY_SPEC.md, and related guides
- **ADRs** — architecture decision records

## What this is NOT

- Scoring code lives in [`brain-wrought-engine`](https://github.com/forgedynamicsai/brain-wrought-engine)
- CLI and runner live in [`brain-wrought-harness`](https://github.com/forgedynamicsai/brain-wrought-harness)
- **Actual judge rubrics** used at eval time are sealed in a private repo. The public scaffolds here describe dimensions without leaking exact prompt wording.

## Why this design

Per [ADR-003](adr/ADR-003-sealed-artifacts.md), Brain-Wrought separates public educational content (here) from sealed evaluation content. Submitters get everything they need to understand and build against — without being able to game the specific prompts used in judging.

## Skill format

Each skill is a Markdown file with this header:

```yaml
---
skill: fixture_generation.clean_brain
version: 1.0.0
model: claude-haiku-4-5  # recommended model for this skill
---
```

Followed by prose instructions for the LLM (or human) executing the skill.

## Contributing

External contributions welcome for:
- Additional fixture templates (new vault organizations, domain-specific brains)
- Query type expansions (new abstention patterns, new personalization dimensions)
- Submitter documentation improvements

NOT accepted:
- Changes to sealed content (maintainer-only)
- Changes that would make rubric scaffolds leak actual judge prompts

## License

MIT.
