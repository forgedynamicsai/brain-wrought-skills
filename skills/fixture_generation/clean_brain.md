---
skill: fixture_generation.clean_brain
version: 1.0.0
model: claude-haiku-4-5
axis: ingestion, retrieval
phase: 1
issue: BW-001
---

# Clean Brain Fixture Generator

Generate a synthetic personal knowledge vault with consistent, well-formed schema. This is the "ideal" brain — used as the baseline fixture tier.

## Structure

A clean brain has 80-150 notes organized as:

```
brain/
├── people/        # one note per person
├── companies/     # one note per organization
├── projects/      # one note per project
├── meetings/      # one note per meeting
└── concepts/      # one note per concept/topic
```

## Note schema (YAML frontmatter required on every note)

```yaml
---
type: person | company | project | meeting | concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [tag1, tag2]
entities: [EntityName1, EntityName2]
state: active | completed | archived  # for projects
---
```

## Content requirements

- **People notes:** name, role, company, relationship context, timeline of interactions, notes
- **Company notes:** name, industry, relationship to user, key contacts (linked), projects (linked)
- **Project notes:** exec summary in first paragraph, status, participants (linked), timeline, decisions
- **Meeting notes:** date, attendees (linked), agenda, decisions, action items
- **Concept notes:** definition, related concepts (linked), user's documented stance

## Link conventions

Use `[[NoteTitle]]` backlinks. Every `[[link]]` must resolve to an existing note. No broken links.

## Generation instructions

Given the entity pool (provided as context), generate notes following the schema above. Vary the content to be plausible — real names, realistic projects, coherent relationships. Do NOT use placeholder text like "Lorem ipsum."

Seed: use `submission_seed + fixture_index` for determinism.
