# HOOX · Report Worker

**Portfolio report generation at the edge — orchestrates a headless Chromium instance inside a Cloudflare Worker to render PDF performance reports.**

[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/) [![Runtime](https://img.shields.io/badge/Runtime-Bun-black?logo=bun)](https://bun.sh) [![Platform](https://img.shields.io/badge/Platform-Cloudflare%C2%AE%20Workers-orange?logo=cloudflare)](https://workers.cloudflare.com/) [![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

**Part of the [HOOX](https://github.com/hoox-sh/hoox) edge-trading mesh — a production-grade algorithmic trading framework on Cloudflare Workers.**  
**Site:** [hoox.sh](https://hoox.sh) · **Docs:** [docs.hoox.sh](https://docs.hoox.sh) · **Paper:** [`hoox-arxiv-paper-core.pdf`](https://github.com/hoox-sh/hoox/blob/main/papers/hoox-arxiv-paper-core.pdf)

---

The report-worker generates PDF portfolio reports using Cloudflare Browser Rendering — a headless Chromium instance launched and controlled from within a V8 isolate. On a scheduled cron trigger (twice-daily configurable interval), it fetches aggregated trade metrics from the [`analytics-worker`](../analytics-worker) and the [`d1-worker`](../d1-worker), renders them as a styled HTML page, and converts the page to PDF via the Browser Rendering API.

Generated reports are persisted to R2 (`trade-reports` bucket) under keyed paths (`reports/daily-{timestamp}.pdf`). On successful generation, the worker fires a notification through the [`telegram-worker`](../telegram-worker) with a download link. The dashboard reads report manifests from R2 for user-facing report history.

### Role in the Mesh

```
analytics-worker ──┐
d1-worker ─────────┼──► report-worker ──► Browser Rendering
                   │        │              (headless Chromium)
                   │        │
                   │        ├──► R2 (reports/daily-*.pdf)
                   │        └──► telegram-worker (PDF ready notification)
                   │
              Scheduled: cron twice-daily
```

### Entry Points

| Trigger     | Path / Event        | Description                         |
| ----------- | ------------------- | ----------------------------------- |
| `scheduled` | Cron (configurable) | Generate and store portfolio report |
| Internal    | Service binding     | On-demand report generation         |

### Data Flow

```
1. Cron tick → collect metrics from analytics-worker + d1-worker
2. Render HTML template with trade history, PnL, Sharpe, drawdown
3. Launch headless Chromium via Browser Rendering API
4. Navigate to generated HTML → `page.pdf()` → Buffer
5. Upload to R2: reports/daily-{timestamp}.pdf
6. Notify telegram-worker: "📊 Daily report ready"
```

### Infrastructure

| Resource                                  | Binding             | Usage                 |
| ----------------------------------------- | ------------------- | --------------------- |
| R2                                        | `REPORTS_BUCKET`    | PDF report storage    |
| Browser Rendering                         | `BROWSER_RENDERING` | Headless Chromium     |
| [`analytics-worker`](../analytics-worker) | `ANALYTICS_SERVICE` | Metric queries        |
| [`d1-worker`](../d1-worker)               | `D1_SERVICE`        | Trade data queries    |
| [`telegram-worker`](../telegram-worker)   | `TELEGRAM_SERVICE`  | Delivery notification |

### Development

```bash
bun test workers/report-worker
```

### License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — part of the HOOX open-core mesh.
