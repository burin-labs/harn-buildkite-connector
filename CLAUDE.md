# CLAUDE.md - harn-buildkite-connector

Pure-Harn connector package for Buildkite Pipelines (CI failure → diagnose / rerun workflows).

Shared Harn connector authoring rules are in the
[connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md).

Keep this file limited to Buildkite-specific notes and local hazards. Put shared connector guidance
in the Harn guide first.

## Provider Notes

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
