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
   nuon healthcheck ensure-trigger            # human-readable
   nuon healthcheck ensure-trigger --json     # machine-readable
   nuon healthcheck ensure-trigger --rotate   # mint a fresh secret on an existing trigger
   ```
   `ensure-trigger` **saves the URL + secret** to a per-org sidecar store
   (`$XDG_CONFIG_HOME/nuon/healthcheck.json`, mode `0600`), so the reporter
   subcommands below just work with no further setup — no exports needed.

   The secret is only shown/stored at create/rotate. To override the store
   (CI, another shell) you can still pass them explicitly — env vars and
   `--url/--secret` always win over the store:
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
# Component may be a NAME (auto-resolved to its id) or a raw cmp… id.
# --check is optional; message/details are optional too.
nuon healthcheck healthy   --component kitchen_sink
nuon healthcheck degraded  --component kitchen_sink --check p99-latency \
  --message "p99 above SLO" --details '{"value_ms":1400}'
```

The status is the subcommand: `healthy` | `degraded` | `unhealthy` | `unknown`.

### Flags

| Flag | Description |
|------|-------------|
| `-c, --component` | Component **name or id** (required; a name is resolved to its id via `nuon components list`) |
| `-n, --check` | Check name (optional; default `$NUON_HEALTH_CHECK`, else `custom-healthcheck`; 1–100 chars, `[a-zA-Z0-9._-]`) |
| `-m, --message` | Human-readable message |
| `-d, --details` | Details object as a JSON string |
| `--via` | `trigger` (default) or `api` |
| `--url` | Trigger ingress URL (trigger mode; or `NUON_HEALTH_TRIGGER_URL`) |
| `--secret` | Trigger HMAC secret (trigger mode; or `NUON_HEALTH_TRIGGER_SECRET`) |
| `-i, --install` | Install id (api mode; defaults to selected install) |
| `--json` | Print the raw response |

## Environment

- Injected by the CLI: `NUON_API_URL`, `NUON_ORG_ID`, `NUON_INSTALL_ID`, `NUON_API_TOKEN`.
- `NUON_HEALTH_CHECK`: default check name when `--check` is omitted (else `custom-healthcheck`).
- Trigger mode: `NUON_HEALTH_TRIGGER_URL`, `NUON_HEALTH_TRIGGER_SECRET`, and
  optionally `NUON_HEALTH_SIG_HEADER` (default `X-Nuon-Signature`),
  `NUON_HEALTH_SIG_PREFIX` (default `sha256=`), `NUON_HEALTH_EVENT_TYPE`
  (default `component.health`).
- Sidecar store: `ensure-trigger` writes the URL + secret to
  `$XDG_CONFIG_HOME/nuon/healthcheck.json` (default `~/.config/nuon/...`, mode
  `0600`, keyed by org id); reporters read it when the env vars/flags are unset.
  Set `NUON_HEALTH_STORE_DIR` to relocate it. The CLI's own `~/.nuon` is never
  touched.

## Requirements

- `curl`, `jq`, and (trigger mode) `openssl` on `PATH`.
- The **component-health** feature enabled for your org.
