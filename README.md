# ding-action

GitHub Actions wrapper for [DING](https://github.com/ding-labs/ding) — alerting that ships with the workload.

Add real-time alerting to any CI step in one line. DING evaluates rules against the JSON events your job emits, fires alerts as GitHub Actions warnings during the run, and writes an aggregate summary to the workflow's step summary when the job exits.

## Usage

```yaml
- uses: ding-labs/ding-action@v1
  with:
    rules: ./alerts.yaml
    command: pytest tests/
```

That's it. The action:

1. Downloads the DING binary for the runner's OS/arch.
2. Wraps your command via `ding run`.
3. Auto-attaches GitHub Actions context (`run_id`, `repo`, `branch`, `commit`, `workflow`, …) to every alert.
4. Surfaces alerts as `::warning::` annotations and renders an aggregate summary in `$GITHUB_STEP_SUMMARY`.
5. Exits with the wrapped command's exit code.

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `command` | yes | — | The command to wrap. Output lines that parse as DING events flow into the rule engine; non-event lines pass through cleanly. |
| `rules` | no | `ding.yaml` | Path to the DING rules file. |
| `version` | no | `latest` | DING version to install (e.g. `v0.6.0`). Pin for reproducibility. |
| `run-id` | no | (auto-detected) | Override the auto-detected run ID. |
| `fail-on-alert` | no | `false` | If `true`, fail the step when any alert fired, even if the wrapped command exited 0. |

## Outputs

| Name | Description |
|---|---|
| `exit-code` | Exit code of the wrapped command. |
| `alerts-fired` | Number of alerts DING fired during the run. |

## A complete example

`alerts.yaml`:

```yaml
rules:
  # Real-time: fires immediately when a test exceeds 5s.
  - name: slow_test
    match: { metric: test.duration }
    condition: value > 5
    message: "slow test {{ .test }} on {{ .branch }}: {{ .value }}s"
    alert: [{ notifier: github_actions }]

  # End-of-run: fires once when pytest exits, with the run's avg latency.
  - name: regression
    match: { metric: test.duration }
    mode: end-of-run
    condition: avg(value) over 1h > 1
    message: "p50 test latency: {{ .avg }}s (count={{ .count }})"
    alert: [{ notifier: github_actions }]

  # Job-level: fires if pytest itself failed.
  - name: pytest_failed
    match: { metric: run.exit }
    condition: value > 0
    message: "pytest exited {{ .value }} after {{ .duration_seconds }}s"
    alert: [{ notifier: github_actions }]
```

`.github/workflows/ci.yml`:

```yaml
name: ci
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install pytest
      - uses: ding-labs/ding-action@v1
        with:
          rules: ./alerts.yaml
          command: pytest tests/ --json-events
```

In your test code, emit JSON events to stdout:

```python
import json, time

def test_something():
    start = time.perf_counter()
    # ... actual test logic ...
    elapsed = time.perf_counter() - start
    print(json.dumps({
        "metric": "test.duration",
        "value": elapsed,
        "test": "test_something"
    }))
```

When `pytest` runs:

- Each slow test surfaces as a yellow GitHub Actions warning in the live log.
- When `pytest` finishes, a markdown table appears in the workflow run's step summary with the regression alert (avg latency over the run) and any job-level alerts (e.g. exit code).
- If pytest exited non-zero, the action exits non-zero too — the check stays red.

## How it differs from a Datadog/New Relic agent in CI

- **No agent installation between runs.** DING is a 5MB binary, downloaded fresh per run, no daemon left behind.
- **No cloud account.** Alerts fire to GitHub Actions step summaries, webhooks, or stdout — wherever you want them.
- **Rules ship with code.** `alerts.yaml` lives in your repo. Reviewable. Versionable. Per-branch.
- **End-of-run summaries.** Datadog can't easily say "the avg of metric X over this CI run was Y" because the run ends before any aggregation rolls up. DING's end-of-run rules are designed for exactly this.

## Supported platforms

The action runs on Linux and macOS GitHub-hosted runners. Windows runners are not yet supported (the underlying DING binary builds for Windows, but the install path in this action assumes a POSIX shell).

If you need Windows support, please open an issue.

## Versioning

This action follows semver tags. Pin to a major version (`@v1`) for automatic minor/patch updates, or pin to an exact tag (`@v1.0.3`) for full reproducibility.

The `version:` input controls which **DING binary** is downloaded — independent of the action's own version.

## License

MIT — same as DING. See [LICENSE](./LICENSE).

## Related

- [DING](https://github.com/ding-labs/ding) — the alerting binary
- [ding.ing](https://ding.ing) — project homepage
