# AGENTS.md

Pure-Harn connector package for Buildkite Pipelines (CI failure → diagnose /
rerun workflows).

Shared connector authoring rules live in the Harn guide:

- [Connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md)

Put shared connector guidance in the Harn guide and keep only
provider-specific notes and local hazards here.

`CLAUDE.md` points here. Edit `AGENTS.md` only.

## Provider notes

- Webhook signing is HMAC-SHA256 over the literal string `"<timestamp>.<raw_body>"` (the unix
  timestamp, a dot, then the raw body), keyed by the webhook **Token**. The header is
  `X-Buildkite-Signature: timestamp=<unix>,signature=<hex>`. Recompute, constant-time compare, then
  separately enforce a ~5-minute freshness window on `timestamp` — the timestamp is inside the
  signed message, so skipping the window check loses replay protection.
- The simpler `X-Buildkite-Token` plain-compare mode is also accepted, but signature mode is
  preferred. When neither a token is configured, inbound is rejected (fail closed).
- There is **no** `build.failed` event. Subscribe to `build.finished` and branch on
  `build.state == "failed"`. `build.failing` is an early mid-build signal, not a terminal state.
- Webhook `build` payloads omit the `jobs` array; use `job.*` events or `build.get` /`api.request`
  against the REST API for per-job data.
- Outbound auth is `Authorization: Bearer <api-token>` against `https://api.buildkite.com/v2`.
  Mutating methods (`job.retry`, `build.rebuild`, `build.cancel`, `job.unblock`) are flagged
  `requires_approval: true` via `methods()`; a retried `job_id` is single-use (use the new id next).
- Do not add compatibility shims or deprecation aliases in this nascent package; cut over directly
  when behavior changes.

<!-- BEGIN HARN SHARED AGENT CONTRACT: managed by harn-bump-fleet -->

## Ecosystem working agreement

- Pursue the ambitious product outcome; make the seams boring with small typed
  interfaces, explicit invariants, and deterministic projections.
- Give each behavior one semantic owner. Generate or parity-test other surfaces
  instead of maintaining competing implementations.
- Work autonomously inside approved scope. Pause for destructive, production,
  high-spend, ambiguous, or authority-expanding actions—not routine reversible work.
- Treat stop, wait, stand down, and pivot as control events for long-lived work.
- Match evidence to the claim. Use the smallest owning product-path check;
  add a falsifier for contested, load-bearing, or potentially vacuous claims.
  Record relevant controls, recovery, and blind spots without repeating proof.
- Evidence follows source and artifact identity, not the branch name. Reuse
  verified branch or merge-candidate evidence after landing when relevant code,
  build inputs, and dependencies are unchanged. Repeat affected checks only
  for a relevant change, observed failure, deployment, or packaging difference.
- "Ship" means integrated on owning main with terminal integration checks and
  applicable release or deployment checks complete. Confirm the landed change
  and merge result; do not rebuild or recapture screenshots solely for main.
- Ship a ready PR by adding the `ship` label when the repo has a Smart Ship
  caller (`.github/workflows/smart-ship.yml`); otherwise land through the merge
  queue with `gh pr merge --squash --auto`. Never `gh pr merge --admin`.
  Incidents use the org override labels `bypass-ci`, `bypass-merge-queue`, or
  `force-merge`.

<!-- END HARN SHARED AGENT CONTRACT -->
