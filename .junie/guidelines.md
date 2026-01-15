# Junie project guidelines (`ai-compute`)

These guidelines are intended for Junie / IDE agents operating inside this repo.

## Project goal (high level)

This repository is a **research + planning workspace** for an "AI compute" idea: identify promising niches, capture customer discovery scripts, and define requirements for dedicated GPU/CPU node offerings.

Primary artifacts live in `docs/` and `general_roadmap.md`.

## Source of truth & where to look first

When answering questions or generating new docs, use these files as context, in this order:

1. `README.md` (if present/updated)
2. `general_roadmap.md` (overall direction)
3. `docs/Top niches.md` (shortlist + wedges)
4. `docs/Research script.md` (how to talk to customers)
5. `docs/List of AI niches where customers often want dedicated nodes.md` (broader market map)
6. `docs/customer-interview-checklists/` (CIC: interview checklists)
7. `docs/niche-requirements-models/` (NRM: niche-by-niche requirements)
8. `competitors/` (competitor feature notes / comparisons)
9. `AllNodes - questions.md` (open questions / backlog)

## How to add new documentation

Prefer **small, atomic additions**:

- If it's a new niche deep-dive: add a new `[NRM][NN][v1] <title>.md` in `docs/niche-requirements-models/`.
- If it's an interview checklist: add a new `[CIC][N][v1] <title>.md` in `docs/customer-interview-checklists/`.
- If it changes strategy/priorities: update `general_roadmap.md` and optionally `docs/Top niches.md`.

### Competitor research

Keep competitor intel lightweight and factual. Prefer capturing:

- what they offer (features / SKUs / locations)
- any explicit constraints (quotas, limits, prerequisites)
- anything that affects wedge positioning (e.g., onboarding friction, security controls, billing rails)

Conventions:

- Store in `competitors/`.
- Prefer one file per competitor (example: `competitors/Oblivus features.md`).
- If you add a new file, start it with `Source:` (link or where the info came from) and `As-of:` (date) when possible.
- If info is missing or uncertain, mark it as `TODO:` rather than guessing.

### Naming conventions

- Keep the existing bracket tags and versioning patterns exactly (e.g. `[NRM][17][v1]...`).
- Titles use normal capitalization and may include separators like `_` when the directory already uses them.

## Writing style

- Be concise, skimmable, and structured with headings and bullet points.
- Optimize for **execution** (what to do next), not essay-style prose.
- When making claims, prefer:
  - explicit assumptions, or
  - references to where the claim comes from (e.g. "from customer call notes"), or
  - a TODO for validation.

## Agent workflow expectations

When asked to create or update docs:

1. Open the most relevant existing doc(s) first.
2. Keep edits minimal and consistent with existing formatting.
3. If a request is ambiguous, ask 1-3 targeted clarifying questions.
4. Don't invent external facts; mark uncertain items as assumptions/TODOs.
