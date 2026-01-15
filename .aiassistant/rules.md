---
apply: always
---

# AI Assistant rules for this repo (`ai-compute`)

These rules are for tooling/agents that use repo documents as context.

## Primary intent

Help maintain and extend a documentation-based project for researching AI compute niches and defining requirements for dedicated nodes.

## Context-loading rules (recommended)

When producing an answer, plan, or new document:

1. Load `general_roadmap.md`.
2. Load `docs/Top niches.md`.
3. If the topic is customer discovery: load `docs/Research script.md` and relevant `docs/customer-interview-checklists/` files.
4. If the topic is a specific niche: load the matching `docs/niche-requirements-models/[NRM]...` file(s).
5. If prioritization or open problems are referenced: load `AllNodes - questions.md`.

If you cannot find a relevant doc, ask the user which niche/checklist/NRM to use.

## Output expectations

- Prefer actionable bullets and checklists.
- Avoid long narratives.
- Use clear section headers.
- Keep scope tight: do not rewrite unrelated documents.

## Truthfulness and uncertainty

- Do not fabricate market numbers, benchmarks, pricing, or company claims.
- If a detail is unknown, label it as `Assumption:` or `TODO:`.
- If referencing external sources, request a link or quote from the user.

## Document maintenance rules

- Preserve existing filename patterns (`[NRM][NN][v1]`, `[CIC][N][v1]`).
- Prefer adding a new version tag (`v2`) only when changes are substantial; otherwise edit in place.
- Keep formatting consistent with neighboring docs.

## Safety / privacy

- Treat any customer data, corp names, or proprietary notes as sensitive.
- When creating examples, use placeholders unless user provides permission.
