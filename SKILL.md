# SKILL: harn-buildkite-connector

Trigger recipes and outbound helpers for Buildkite via the pure-Harn
`harn-buildkite-connector` package. Receive a webhook on CI failure,
map it to the PR / commit / branch, then let an agent diagnose or rerun.

## What you get

- **Signed webhook inbound** — HMAC-SHA256 over `"<timestamp>.<raw_body>"`
  with a constant-time compare and a ~5-minute freshness window
  (replay protection). A simpler `X-Buildkite-Token` plain-compare mode
  is also supported. Fails closed when no token is configured.
- **Normalized CI envelope** for `build.finished`, `job.finished`, and
  the rest of the `build.*` / `job.*` family, with `is_failure`,
  `commit_sha`, `branch`, `pr_number`, and a `rerun_handle`.
- **REST outbound** for retrying jobs, rebuilding/canceling builds,
  fetching the build (with its jobs array), tailing job logs, and
  unblocking blocked jobs.
- **GraphQL passthrough** and a raw `api.request` escape hatch.
- **GitHub PR resolution** helper to map a commit SHA to its PRs when
  `build.pull_request` is absent.

## There is no `build.failed` event

Buildkite emits `build.finished` for every terminal build. Branch on
`build.state == "failed"` (the connector also re-tags the normalized
`kind` to `build.failed` / `build.passed` for convenience). The states
are `scheduled`, `running`, `passed`, `failed`, `blocked`, `canceled`,
`canceling`, `skipped`, `not_run`, `waiting`, and `waiting_failed`.
`build.failing` is an early mid-build signal, not a terminal state.

## Trigger recipe — diagnose a failed build

```harn
import buildkite_connector from "harn-buildkite-connector"

trigger ci_failure on buildkite {
  source = {
    kind: "webhook",
    events: ["build.finished"],
  }
  on event {
    if event.payload.is_failure {
      // Webhook `build` omits the jobs array — fetch it.
      let build = buildkite_connector.call("build.get", {
        org: event.payload.rerun_handle.org,
        pipeline: event.payload.pipeline_slug,
        build_number: event.payload.build_number,
        api_token: env("BUILDKITE_API_TOKEN"),
      })
      // Hand `build.jobs`, `event.payload.commit_sha`, and the PR to an agent.
    }
  }
}
```

## Trigger recipe — retry the first failed job (mutating)

`job.retry`, `build.rebuild`, `build.cancel`, and `job.unblock` are
flagged `requires_approval: true` via `methods()`, so the orchestrator
gates them behind the standard approval boundary.

```harn
trigger autoretry on buildkite {
  source = { kind: "webhook", events: ["job.finished"] }
  on event {
    let job = event.payload.job
    if job != nil && job.exit_status != 0 {
      // A job_id is single-use: retrying returns a NEW job_id.
      buildkite_connector.call("job.retry", {
        org: event.payload.rerun_handle.org,
        pipeline: event.payload.pipeline_slug,
        build_number: event.payload.build_number,
        job_id: job.id,
        api_token: env("BUILDKITE_API_TOKEN"),
      })
    }
  }
}
```

## Resolve the PR when `build.pull_request` is absent

```harn
let pulls = buildkite_connector.call("github_commit_pulls", {
  owner: "acme",
  repo: "web",
  sha: event.payload.commit_sha,
  github_token: env("GITHUB_TOKEN"),
})
// pulls[0].number is the PR for this commit, if any.
```

## Required secrets per binding

| Secret                          | Used for                                          |
| ------------------------------- | ------------------------------------------------- |
| `buildkite/webhook-token`       | Verify inbound webhooks (signature or token mode) |
| `buildkite/api-token`           | Outbound REST + GraphQL as `Authorization: Bearer`|

Scope the API token `read_builds` (for `build.get` / `job.log`) and
`write_builds` (for retry / rebuild / cancel / unblock).

## Self-managed / custom API host

Set `api_base_url` on each call (or `BUILDKITE_API_BASE_URL`) to target
a non-default host; the default is `https://api.buildkite.com/v2`.
