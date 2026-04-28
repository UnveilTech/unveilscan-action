# Unveilscan GitHub Action

[![Marketplace](https://img.shields.io/badge/marketplace-unveilscan-blue?logo=github)](https://github.com/marketplace/actions/unveilscan)

Run a [Unveilscan](https://unveilscan.com) passive security audit on every
push or schedule and fail the build if the severity reaches a threshold
you set.

**89 checks** across DNS, TLS, web headers, email auth, public leaks,
known CVEs (Shodan + OSV), CT-log surveillance, compliance mapping
(PCI-DSS 4.0 / ISO 27001:2022 / SOC 2 / NIS 2 / GDPR). 100 % passive on
the Basic + Extended profiles — no fuzzing, no exploitation. EU-based,
EUR-priced, free tier with public score badges.

## Quickstart

1. Create an account at https://unveilscan.com (signup is free).
2. For Extended / Active profiles, verify your domain (DNS TXT or
   `.well-known/` file). Basic scans are open-domain.
3. Create a token at **Settings → Tokens** and store it as a repo
   secret named `UNVEILSCAN_TOKEN`.
4. Add the workflow below.

```yaml
name: Security scan
on:
  schedule:
    - cron: '0 6 * * *'           # daily 06:00 UTC
  workflow_dispatch:

jobs:
  unveilscan:
    runs-on: ubuntu-latest
    steps:
      - uses: unveiltech/unveilscan-action@v1
        with:
          domain: example.com
          token:  ${{ secrets.UNVEILSCAN_TOKEN }}
          profile: extended
          fail-on: high
          output: json
```

## Inputs

| Input             | Default                | Description                                                                 |
| ----------------- | ---------------------- | --------------------------------------------------------------------------- |
| `domain`          | _required_             | Domain to scan.                                                             |
| `token`           | _required_             | Bearer token (`UNVEILSCAN_TOKEN` secret).                                   |
| `profile`         | `basic`                | `basic` / `extended` / `active`.                                            |
| `fail-on`         | `high`                 | Min severity to exit non-zero: `info`/`low`/`medium`/`high`/`critical`.     |
| `output`          | `text`                 | `text` / `json` / `csv`.                                                    |
| `timeout`         | `10m`                  | Overall deadline (Go duration).                                             |
| `ack-active`      | `false`                | Set `true` when `profile: active` (ack intrusive probes).                   |
| `include-ssllabs` | `false`                | Enable opt-in Qualys SSL Labs deep TLS pass.                                |
| `base-url`        | `https://unveilscan.com` | Override for self-hosted Unveilscan instances.                            |
| `cli-version`     | _action ref_           | Pin the CLI version. Default tracks the action tag (`@v1` ⇒ latest v1.x).  |

## Exit codes

| Code | Meaning                                          |
| ---- | ------------------------------------------------ |
| `0`  | Scan completed, no finding ≥ `fail-on`.          |
| `1`  | At least one finding ≥ `fail-on`.                |
| `2`  | API error, auth error, network error, timeout.   |
| `3`  | Invalid argument.                                |

## Recipes

### Scan multiple domains in parallel

```yaml
strategy:
  matrix:
    domain: [app.example.com, api.example.com, www.example.com]
steps:
  - uses: unveiltech/unveilscan-action@v1
    with:
      domain: ${{ matrix.domain }}
      token:  ${{ secrets.UNVEILSCAN_TOKEN }}
```

### Keep the JSON report as an artifact

```yaml
- uses: unveiltech/unveilscan-action@v1
  with:
    domain: example.com
    token:  ${{ secrets.UNVEILSCAN_TOKEN }}
    output: json
  id: scan
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: unveilscan-${{ github.run_id }}
    path: unveilscan.json
```

### Block PRs that introduce a regression

```yaml
on: [pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: unveiltech/unveilscan-action@v1
        with:
          domain: staging.example.com
          token:  ${{ secrets.UNVEILSCAN_TOKEN }}
          profile: extended
          fail-on: medium      # tighter for PR gates
```

## Architecture support

The action auto-detects `linux/amd64` and `linux/arm64`. Other runners
(macOS, Windows) are not currently supported — open an issue if you
need them.

## Self-hosted Unveilscan

Pass `base-url: https://your-instance.example.com`. The CLI binary is
still pulled from this action's GitHub releases (so you don't need to
expose a download endpoint on your side).

## License

[MIT](LICENSE) for the action wrapper. The Unveilscan service itself is
proprietary; the CLI binary distributed via this action is licensed
separately, see https://unveilscan.com/docs.
