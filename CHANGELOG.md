# Changelog

All notable changes to `harn-buildkite-connector` will be documented in
this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Initial pure-Harn Buildkite connector implementing the connector
  interface (`provider_id`, `kinds`, `methods`, `payload_schema`,
  lifecycle, `normalize_inbound`, `call`).
- Signed webhook verification: HMAC-SHA256 over `"<timestamp>.<raw_body>"`
  keyed by the webhook token, constant-time compare, plus a ~5-minute
  freshness window for replay protection. Simpler `X-Buildkite-Token`
  plain-compare mode is also supported, and inbound fails closed when no
  token is configured.
- Normalization of `build.*` / `job.*` events into a canonical CI
  envelope (`provider`, `kind`, `state`, `is_failure`, `commit_sha`,
  `branch`, `pr_number`, `pipeline_slug`, `build_number`, `build_id`,
  `web_url`, `job`, `rerun_handle`). `build.finished` is re-tagged to
  `build.failed` / `build.passed` since Buildkite has no `build.failed`
  event. Dedup is on `(pipeline.slug, build.number, event, build.state)`.
- Outbound `call(method, args)` dispatch: `build.get`, `build.rebuild`,
  `build.cancel`, `job.retry`, `job.unblock`, `job.log`, a raw
  `api.request` escape hatch, a `graphql` helper, and an optional
  `github_commit_pulls` PR-by-SHA resolver. Mutating methods are flagged
  `requires_approval` via `methods()`.
- HTTPS-only outbound with SSRF host guards, and
  `RateLimit-Remaining` / `RateLimit-Reset` handling with a 60-second cap
  on sleep-and-retry.
- Signed-fixture smoke tests covering normalization mapping, signature
  pass/fail (tampered, wrong token, missing token), stale-timestamp
  rejection, the dedup key, and a mock-HTTP call smoke.

[Unreleased]: https://github.com/burin-labs/harn-buildkite-connector/compare/main
