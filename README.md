# Healkite GitHub Action

Ship over-the-air updates for React Native with [Healkite](https://healkite.com) from GitHub Actions:
publish (guarded or scheduled) releases, guard bundle size on pull requests, and open **PR previews** that
testers try inside the store build.

Store `HEALKITE_TOKEN` (an access key from the dashboard → API & MCP; *Developer* role) and, if your app
requires signed releases, `HEALKITE_SIGNING_KEY` (your PEM) as repository secrets.

## Release on merge

```yaml
on: { push: { branches: [main] } }
jobs:
  ota:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - uses: saif-09/healkite-action@v1
        with:
          api-url: https://api.healkite.com
          token: ${{ secrets.HEALKITE_TOKEN }}
          signing-key: ${{ secrets.HEALKITE_SIGNING_KEY }}
          channel: production-xxxxxxxx
          platform: android
          runtime-version: "1.0"
          guarded: "true"          # 1% → 10% → 50% → 100%, halts on crashes
          max-growth: "10"         # refuse bundles that grew > 10%
```

## Bundle size check on pull requests

```yaml
      - uses: saif-09/healkite-action@v1
        with:
          mode: size-check
          api-url: https://api.healkite.com
          token: ${{ secrets.HEALKITE_TOKEN }}   # Viewer role is enough
          channel: production-xxxxxxxx
          platform: android
          runtime-version: "1.0"
          max-growth: "10"
```

## PR previews on the store build

```yaml
on:
  pull_request: { types: [opened, synchronize, reopened, closed] }
permissions: { pull-requests: write, contents: read }
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
        if: github.event.action != 'closed'
      - uses: saif-09/healkite-action@v1
        with:
          mode: ${{ github.event.action == 'closed' && 'preview-close' || 'preview' }}
          api-url: https://api.healkite.com
          token: ${{ secrets.HEALKITE_TOKEN }}
          signing-key: ${{ secrets.HEALKITE_SIGNING_KEY }}
          channel: production-xxxxxxxx   # the channel your store build uses
          platform: android
          runtime-version: "1.0"         # the store build testers have
```

The action comments a link on the PR; testers open it on their phone (or scan the QR code) and the installed
app switches to the PR's code with an exit banner. Your app needs a URL scheme set in the dashboard and
`Healkite.openPreview(url)` wired to incoming links.

## Inputs

| Input | |
|---|---|
| `mode` | `release` (default), `size-check`, `preview`, `preview-close` |
| `api-url`, `token`, `channel`, `platform`, `runtime-version` | Required |
| `signing-key` | PEM of your ECDSA P-256 key (secret) |
| `guarded` | `"true"` for a guarded rollout |
| `description` | Release notes |
| `starts-at`, `ends-at` | Schedule a release / campaign (`+2h`, `2026-12-24 00:00`, ISO) |
| `max-size`, `max-growth` | Size budgets (`2MB`, `10`) |
| `preview-name` | Default `pr-<number>` |
| `project` | React Native project directory (default `.`) |

Output: `size-summary` (e.g. `Bundle 520.0 KB (+20.0 KB, +4.0% vs v4)`).
