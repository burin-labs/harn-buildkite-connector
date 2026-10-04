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

- Build ambitious outcomes behind small typed interfaces; give behavior one owner
  and generate or parity-test projections instead of duplicating policy.
- Work autonomously within approved scope. Pause for destructive or production effects,
  exceptional spend, material ambiguity, or new authority.
- Treat stop, wait, stand down, pivot, and steer as control events.
- Use the smallest owning product-path check. Add a falsifier for contested, load-bearing,
  or potentially vacuous claims; record controls, recovery, and blind spots.
- Evidence follows source/artifact identity. Reuse proof when relevant code, build inputs,
  and dependencies are unchanged. Repeat affected checks for relevant changes, failures,
  deployment, or packaging differences. Do not rebuild or recapture solely for main.
- Ship means owning-main integration with terminal merge and applicable release/deploy
  checks. Confirm landed content and result; an open PR is incomplete.
- Use `ship` with a deployed Smart Ship caller; otherwise use `gh pr merge --squash --auto`.
  Never use `--admin`; incidents use `bypass-ci`, `bypass-merge-queue`, or `force-merge`.

<!-- END HARN SHARED AGENT CONTRACT -->
