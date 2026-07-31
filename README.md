# nuon-ext-healthcheck

A Nuon CLI extension for reporting **custom component health checks** to the
Nuon API. It wraps the

```
PUT /v1/installs/{install_id}/components/{component_id}/health/checks/{check_name}
```

endpoint so you can push a named health signal (from CI, a monitor webhook, a
custom action, etc.) using your active `nuon` context instead of hand-rolled
`curl` with hardcoded IDs and tokens.

## Install

```bash
# local directory (symlink — best for development)
nuon ext install ./nuon-ext-healthcheck

# once published
nuon ext install nuonco/nuon-ext-healthcheck
```

## Usage

```bash
nuon healthcheck set \
  --component cmp43ei... \
  --check my-check \
  --status degraded \
  --message "p99 latency above SLO (1.4s > 800ms)" \
  --details '{"metric":"http_request_duration_p99","value_ms":1400,"threshold_ms":800}'
```

Status shorthands:

```bash
nuon healthcheck healthy   --component cmp... --check my-check --message "within SLO"
nuon healthcheck degraded  --component cmp... --check my-check --message "elevated latency"
nuon healthcheck unhealthy --component cmp... --check my-check --message "returning 5xx"
nuon healthcheck unknown   --component cmp... --check my-check --message "metrics unreachable"
```

### Flags

| Flag              | Description                                                              |
| ----------------- | ------------------------------------------------------------------------ |
| `-c, --component` | Component id (required)                                                  |
| `-n, --check`     | Check name (required; 1–100 chars, `[a-zA-Z0-9._-]`, alphanumeric ends)  |
| `-s, --status`    | `healthy` \| `degraded` \| `unhealthy` \| `unknown` (required for `set`) |
| `-m, --message`   | Human-readable message                                                   |
| `-d, --details`   | Details object as a JSON string, e.g. `'{"value_ms":1400}'`              |
| `--stale-after`   | How long the report stays trusted (e.g. `30m`; default 5m, max 60m)      |
| `-i, --install`   | Install id (defaults to the selected install)                            |
| `--json`          | Print the raw API response                                               |

## Context

The Nuon CLI injects these automatically from the active context; you do not set
them by hand:

- `NUON_API_URL`, `NUON_ORG_ID`, `NUON_INSTALL_ID`, `NUON_API_TOKEN`

## Requirements

- `curl` and `jq` on `PATH`.
- The **component-health** feature enabled for your org.
