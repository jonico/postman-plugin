---
name: ai-readiness
description: Scores a Postman collection or an OpenAPI spec for how well an AI agent can discover, understand, call, and recover from errors with it — missing examples, undocumented errors, and ambiguous parameters all cost points. Use when the user asks "is my API agent-ready," "can AI agents use my API," "how agent-friendly is my API," or wants to scan, score, or improve a collection or spec for AI/agent consumption. Covers `postman collection ai-readiness` and `postman spec ai-readiness`.
---

# AI Readiness

## Overview

An "agent-ready" API is one that an AI agent can discover, understand, call correctly, and recover from errors without human intervention. Most APIs aren't there yet.

Two ways to run this check, same rubric family, different target — pick by what exists:

- `collection ai-readiness <collectionId/path>` scores a Postman collection
  — by cloud ID, local file path, or a `postman/collections/<name>`
  local-mode directory.
- `spec ai-readiness <spec>` scores an OpenAPI specification directly — by
  cloud ID or local file path — with no collection involved at all.



## Scoring

- **Critical checks (4x weight):** Blocks agent usage entirely
- **High checks (2x weight):** Causes frequent agent failures
- **Medium checks (1x weight):** Degrades agent performance
- **Low checks (0.5x weight):** Nice-to-have improvements

**Agent Ready = score of 70% or higher with zero critical failures.**

## Interpreting Results

- **90-100%:** Excellent. Agents can use this API reliably.
- **70-89%:** Agent-ready. Minor improvements possible.
- **50-69%:** Not agent-ready. Key issues need fixing.
- **Below 50%:** Significant work needed. Focus on critical failures first.

State the actual score and verdict the command printed, which verb ran
(`collection ai-readiness` vs. `spec ai-readiness`), which target was
scored (local path vs. cloud ID) and which output mode was used, and — if
`--min-score` was set — the resulting exit code, not just "it passed."

You can ask user if they would like to set this check with a min score guarantee to run on their CI.

## Reference

- `collection-schema-v3` skill — what saved examples and descriptions look
  like in the git-synced format this command reads.
- `ci-integration` skill — where `--min-score` fits as a pipeline gate
  alongside `spec lint`/`collection lint`/`workspace lint`.
