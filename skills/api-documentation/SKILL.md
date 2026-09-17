---
name: api-documentation
description: Publishes Postman's auto-generated API documentation, rendered from an OpenAPI 3.0 definition or a collection rather than hand-written. Use when the user asks to "publish API docs", "generate documentation for this API", "put this on the API Network", or "share a docs link for this collection or spec". Distinct from Documents (freeform collaborative notes) and Spec Hub (the design and authoring surface). Requires bootstrap.
disable-model-invocation: true
---

# Publish API Documentation

## Overview

Postman auto-generates browsable API documentation from an OpenAPI 3.0
definition or from a collection — it is not hand-written prose. This is a
different feature from Documents (freeform notes alongside a workspace) and
Spec Hub (where the spec itself is authored); don't conflate them. Requires
`bootstrap`'s resolved spec/collection path.

## Critical Rules

1. **Docs are generated from the spec or collection, never hand-written in
   parallel.** A field with no description is a gap to fix at the source,
   not in a doc written around it — a hand-maintained page drifts from the
   contract that actually ships; generation is what keeps them equal.
   Publish the source and Postman renders the docs from it automatically:
   `postman workspace push` for a git-native spec/collection. There is no CLI
   command literally named "generate docs" — confirm the verb with `postman
   workspace -h` rather than guessing one.
   `postman api publish <apiId>` is **not** the second option it looks like:
   it publishes an API Builder object, which Postman's docs call *"deprecated
   and no longer supported"* and *"no longer supported in Postman v12 and
   later"*, and it is US-region-only. Use it only against an API that already
   lives there, and say it should move to Spec Hub.
2. **Publishing externally requires explicit consent; regenerating a
   local/private preview does not.** External means the Postman API Network
   or a public domain — either makes the docs reachable outside the team.
3. **State which source rendered the docs when both a spec and a collection
   exist.** They can disagree; say which one the reader is looking at.
4. **Run `-h` before any `postman` command not spelled out above.**
   `postman <resource> <action> -h` prints the real actions, flags and
   defaults — copy the shape from its output. Never substitute a verb that
   sounds right; a wrong one fails as if the feature were missing.

## Verification

- The published content traces back to the spec/collection, not to prose
  written specifically for the doc page.
- External publish only happened after explicit consent was given.
- The published URL was returned and confirmed to resolve. A successful
  publish call is not the same as a reachable page.
