# harn-buildkite-connector

Pure-Harn [Buildkite](https://buildkite.com/) connector for the Harn
orchestrator. Verifies signed inbound webhooks, normalizes Buildkite
`build.*` / `job.*` events into a canonical CI envelope, and dispatches
outbound REST / GraphQL calls so an agent can diagnose or rerun a failed
build.

This is a first-party **inbound + outbound** connector package implementing
Harn Connector Contract v1.

## User story

Receive a failed-build webhook, map it to the PR, commit, and branch, then let
an agent diagnose the failure or rerun the build.

## Install

No release tag is published yet. Use the `main` branch for package-manager
smoke tests:

```sh
harn add github.com/burin-labs/harn-buildkite-connector@main
```

Use a path checkout for local multi-repo development:

```toml
[dependencies]
harn-buildkite-connector = { path = "../harn-buildkite-connector" }
```

## Usage

### Webhook trigger: diagnose a failed build

```harn
import buildkite_connector from "harn-buildkite-connector"

trigger ci_failure on buildkite {
  source = {
    kind: "webhook",
    events: ["build.finished"],
  }
  on event {
    // There is no `build.failed` event — branch on the state.
    if event.payload.is_failure {
      let build = buildkite_connector.call("build.get", {
        org: event.payload.rerun_handle.org,
        pipeline: event.payload.pipeline_slug,
        build_number: event.payload.build_number,
        api_token: env("BUILDKITE_API_TOKEN"),
      })
      // Hand build.jobs + event.payload.commit_sha + the PR to an agent.
    }
  }
}
```

### Retry a job

```harn
buildkite_connector.call("job.retry", {
  org: "acme",
  pipeline: "web",
  build_number: 43,
  job_id: "j5555555-...",
  api_token: env("BUILDKITE_API_TOKEN"),
})
// A retried job_id is single-use; the response carries the NEW job_id.
```

## Normalized event envelope

`normalize_inbound` returns the canonical tagged `NormalizeResult`
(`{type: "event" | "reject", ...}`). The event payload is:

```text
{
  provider: "buildkite",
  kind,                  // build.finished -> re-tagged build.failed / build.passed
  state,                 // raw build.state
  is_failure,            // true when state is failed / canceled / waiting_failed
  repo,                  // pipeline.repository
  commit_sha,            // build.commit
  branch,                // build.branch
  pr_number,             // build.pull_request.id, when PR-triggered
  pipeline_slug,
  build_number,
  build_id,
  web_url,
  job,                   // present for job.* events
  rerun_handle: { org, pipeline, build_number, job_id? },
  raw,                   // the full upstream payload
}
```

The dedupe key is the tuple `(pipeline.slug, build.number, event,
build.state)`, since Buildkite webhooks carry no per-delivery id.

## Webhook verification

Buildkite signs webhooks with HMAC-SHA256. The
`X-Buildkite-Signature: timestamp=<unix>,signature=<hex>` header carries
the HMAC of the literal string `"<timestamp>.<raw_body>"`, keyed by the
webhook **Token**. The connector recomputes the HMAC, compares it in
constant time, and — because the timestamp is part of the signed message
— also enforces a ~5-minute freshness window to defeat replays.

A simpler `X-Buildkite-Token` plain-compare mode is also accepted, but
signature mode is preferred. When **no** webhook token is configured,
inbound is rejected (fail closed).

## Outbound methods

| Method                 | HTTP                                                              | Approval |
| ---------------------- | ---------------------------------------------------------------- | -------- |
| `build.get`            | `GET .../builds/{number}` (returns the jobs array)               | no       |
| `job.log`              | `GET .../jobs/{job_id}/log`                                       | no       |
| `job.retry`            | `PUT .../jobs/{job_id}/retry`                                     | yes      |
| `build.rebuild`        | `PUT .../builds/{number}/rebuild`                                 | yes      |
| `build.cancel`         | `PUT .../builds/{number}/cancel`                                  | yes      |
| `job.unblock`          | `PUT .../jobs/{job_id}/unblock`                                   | yes      |
| `api.request`          | raw `{method, path, body}` escape hatch                          | no       |
| `graphql`              | `POST https://graphql.buildkite.com/v1`                          | no       |
| `github_commit_pulls`  | `GET /repos/{owner}/{repo}/commits/{sha}/pulls` (resolve PR)     | no       |

`methods()` exposes the `requires_approval` flag per method; the
orchestrator gates mutating calls behind its approval boundary.

The REST base is `https://api.buildkite.com/v2` with
`Authorization: Bearer <buildkite/api-token>`. Override per call with
`api_base_url` or via `BUILDKITE_API_BASE_URL`.

## Authentication

| Secret                    | Used for                                            |
| ------------------------- | --------------------------------------------------- |
| `buildkite/webhook-token` | Inbound webhook verification (signature/token mode) |
| `buildkite/api-token`     | Outbound REST + GraphQL Bearer auth                 |

For REST helpers, scope the API token with `read_builds` for `build.get`,
`read_build_logs` for `job.log`, and `write_builds` for retry, rebuild, cancel,
and unblock. GraphQL calls also require Buildkite's separate GraphQL API access
capability; REST scopes do not limit GraphQL access.

## Development

Run the package gate:

```sh
harn connector test "$(pwd)" --provider buildkite
```

## License

Dual-licensed under MIT and Apache-2.0.

- [LICENSE-MIT](./LICENSE-MIT)
- [LICENSE-APACHE](./LICENSE-APACHE)
