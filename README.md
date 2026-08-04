# nuon-ext-healthcheck

A Nuon CLI extension for reporting **custom component health checks**.

By default it emits a health **event** to a Nuon **trigger** ingress URL; a
trigger rule routes the event to a runbook that records the health check. This
decouples reporters from the internal API and flows every report through Nuon's
event pipeline (auth, filtering, audit, replay), so the CLI, a Datadog monitor,
CI, etc. can all feed the same trigger. A legacy `--via api` mode still writes
the health-check API directly.

## Install

```bash
nuon ext install ./nuon-ext-healthcheck      # local (symlink)
nuon ext install nuonco/nuon-ext-healthcheck  # once published
```

## Setup (trigger mode)

Two commands — one provisions the trigger (API), one emits the rule to sync:

1. **Create the trigger** (idempotent by name; prints the ingress URL + secret):
   ```bash
   nuon healthcheck create-trigger            # human-readable
   nuon healthcheck create-trigger --json     # for stashing the secret
   nuon healthcheck create-trigger --rotate   # mint a fresh secret on an existing trigger
   ```
   The secret is only shown at create/rotate. Then export what it prints:
   ```bash
   export NUON_HEALTH_TRIGGER_URL='https://.../v1/event-ingress/<key>'
   export NUON_HEALTH_TRIGGER_SECRET='<secret>'
   ```

2. **Generate the rule** for the selected install, then paste it into
   `triggers.toml` and `nuon sync`:
   ```bash
   nuon healthcheck generate-rule                       # uses selected install
   nuon healthcheck generate-rule --install <install-id>
   ```
   **Rules can't be created via the API** (they're pinned to app-config
   versions), so this prints the `[[triggers.rules]]` block for you to commit +
   sync. The `report-health` runbook (`runbooks/report-health/`) must be synced too.

### How routing works

A trigger rule can't pick the target install from the payload — `target.install`
is a static **name**. So `generate-rule` resolves the selected install's name
from its id and emits a rule that (a) filters on `$.install == <id>` and
(b) targets that install by name. The install to report for still travels in the
payload (`$.install`, an id), is mapped to a runbook input, and the runbook uses
it in the API path (`/v1/installs/{install}/...`). Run `generate-rule` once per
install you want to cover.

## Usage

```bash
nuon healthcheck degraded \
  --component cmp43ei... --check my-check \
  --message "p99 above SLO" --details '{"value_ms":1400}'

nuon healthcheck healthy --component cmp... --check my-check
```

The status is the subcommand: `healthy` | `degraded` | `unhealthy` | `unknown`.

### Flags

| Flag | Description |
|------|-------------|
| `-c, --component` | Component id (required) |
| `-n, --check` | Check name (required; 1–100 chars, `[a-zA-Z0-9._-]`) |
| `-m, --message` | Human-readable message |
| `-d, --details` | Details object as a JSON string |
| `--via` | `trigger` (default) or `api` |
| `--url` | Trigger ingress URL (trigger mode; or `NUON_HEALTH_TRIGGER_URL`) |
| `--secret` | Trigger HMAC secret (trigger mode; or `NUON_HEALTH_TRIGGER_SECRET`) |
| `-i, --install` | Install id (api mode; defaults to selected install) |
| `--json` | Print the raw response |

## Environment

- Injected by the CLI: `NUON_API_URL`, `NUON_ORG_ID`, `NUON_INSTALL_ID`, `NUON_API_TOKEN`.
- Trigger mode: `NUON_HEALTH_TRIGGER_URL`, `NUON_HEALTH_TRIGGER_SECRET`, and
  optionally `NUON_HEALTH_SIG_HEADER` (default `X-Nuon-Signature`),
  `NUON_HEALTH_SIG_PREFIX` (default `sha256=`), `NUON_HEALTH_EVENT_TYPE`
  (default `component.health`).

## Requirements

- `curl`, `jq`, and (trigger mode) `openssl` on `PATH`.
- The **component-health** feature enabled for your org.
