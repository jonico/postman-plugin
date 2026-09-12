---
name: monitoring
description: This skill should be used when the user asks to "monitor this API in production", "set up a scheduled check against the live endpoint", "alert us if the deployed API breaks", or "catch contract regressions between deploys". Covers Postman Monitors — a scheduled check against a deployed environment, distinct from the ci skill's per-push check against a freshly built one. `postman monitor run` only invokes an existing Monitor and reports the result; there is no CLI command that creates or schedules one.
---

# Monitor the Live Endpoint

## Overview

A scheduled check against a deployed environment, using the same collection
a human runs locally and CI runs on push. Monitors catch regressions that
appear between deploys — a dependency changing behavior, a cert expiring,
data drift — not at deploy time. Requires `bootstrap`'s resolved collection
and environment.

## Critical Rules

1. **Reuse the existing collection and environment.** Never duplicate
   requests or assertions into monitor-only config.
2. **Never point a monitor at production without explicit consent.** A
   Monitor runs on Postman's infrastructure on a recurring schedule and can
   alert real people — confirm the target environment and alert destination
   with the user before creating one. Creation and scheduling happen in the
   Postman app or API, not the CLI — the CLI's only monitor verb is
   `postman monitor run <monitorId>` (`-t/--timeout`, default 15 min), which
   invokes a Monitor that already exists and reports the result.
3. **Don't invent an alert destination.** Email, Slack channel, webhook —
   whatever the user gives. If they haven't said, ask; don't default to
   nothing meaningful or guess an address.
4. **State the frequency tradeoff instead of picking a number silently.**
   More frequent checks catch regressions faster and spend more monitor
   runs — say what's being chosen and why.

## Verification

- The monitor references the same collection id used locally and in CI —
  not a copy.
- Target environment and alert destination were both explicitly confirmed,
  not assumed.
