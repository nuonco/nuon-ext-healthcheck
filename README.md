# nuon-ext-healthcheck

A Nuon CLI extension for reporting **custom component health checks**.

It PUTs the check straight to the Nuon health-check API:

```
PUT /v1/installs/{install}/components/{component}/health/checks/{check}
```

No triggers, runbooks, or event pipeline — the reporter talks to the API directly.

## Install

```bash
nuon ext install ./nuon-ext-healthcheck      # local (symlink)
nuon ext install nuonco/nuon-ext-healthcheck  # once published
```

## Usage

```bash
# Component may be a NAME (auto-resolved to its id) or a raw cmp… id.
# --check is optional; message/details are optional too.
nuon healthcheck healthy   --component kitchen_sink
nuon healthcheck degraded  --component kitchen_sink --check p99-latency \
  --message "p99 above SLO" --details '{"value_ms":1400}'
```

The status is the subcommand: `healthy` | `degraded` | `unhealthy` | `unknown`.
The install defaults to the selected install (`NUON_INSTALL_ID`); pass
`--install` to report for another.

### Flags

| Flag | Description |
|------|-------------|
| `-c, --component` | Component **name or id** (required; a name is resolved to its id via `nuon components list`) |
| `-n, --check` | Check name (optional; default `$NUON_HEALTH_CHECK`, else `custom-healthcheck`; 1–100 chars, `[a-zA-Z0-9._-]`) |
| `-m, --message` | Human-readable message |
| `-d, --details` | Details object as a JSON string |
| `-i, --install` | Install id (defaults to selected install) |
| `--json` | Print the raw response |
| `--curl` | Print the equivalent `curl` command instead of sending (token/org emitted as `$NUON_API_TOKEN`/`$NUON_ORG_ID` refs, not literals) |

## Environment

- Injected by the CLI: `NUON_API_URL`, `NUON_ORG_ID`, `NUON_INSTALL_ID`, `NUON_API_TOKEN`.
- `NUON_HEALTH_CHECK`: default check name when `--check` is omitted (else `custom-healthcheck`).

## Requirements

- `curl` and `jq` on `PATH`.
- The **component-health** feature enabled for your org.
