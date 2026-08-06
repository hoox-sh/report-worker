# HOOX · Report Worker

**Portfolio report generation at the edge — orchestrates headless Chromium (Browser Rendering) to produce PDF performance reports.**

**Portfolio report generation at the edge — orchestrates a headless Chromium instance inside a Cloudflare Worker to render PDF performance reports.**

[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/) [![Runtime](https://img.shields.io/badge/Runtime-Bun-black?logo=bun)](https://bun.sh) [![Platform](https://img.shields.io/badge/Platform-Cloudflare%C2%AE%20Workers-orange?logo=cloudflare)](https://workers.cloudflare.com/) [![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

**Part of the [HOOX](https://github.com/hoox-sh/hoox) edge-trading mesh — a production-grade algorithmic trading framework on Cloudflare Workers.**  
**Site:** [hoox.sh](https://hoox.sh) · **Docs:** [docs.hoox.sh](https://docs.hoox.sh) · **Paper:** [`hoox-arxiv-paper-core.pdf`](https://github.com/hoox-sh/hoox/blob/main/papers/hoox-arxiv-paper-core.pdf)

---

The report-worker generates PDF portfolio reports using Cloudflare Browser Rendering — a headless Chromium instance launched and controlled from within a V8 isolate. On a scheduled cron trigger (twice-daily configurable interval), it fetches aggregated trade metrics from the [`analytics-worker`](https://github.com/hoox-sh/analytics-worker) and the [`d1-worker`](https://github.com/hoox-sh/d1-worker), renders them as a styled HTML page, and converts the page to PDF via the Browser Rendering API.

Generated reports are persisted to R2 (`trade-reports` bucket) under keyed paths (`reports/daily-{timestamp}.pdf`). On successful generation, the worker fires a notification through the [`telegram-worker`](https://github.com/hoox-sh/telegram-worker) with a download link. The dashboard reads report manifests from R2 for user-facing report history.

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
| [`analytics-worker`](https://github.com/hoox-sh/analytics-worker) | `ANALYTICS_SERVICE` | Metric queries        |
| [`d1-worker`](https://github.com/hoox-sh/d1-worker)               | `D1_SERVICE`        | Trade data queries    |
| [`telegram-worker`](https://github.com/hoox-sh/telegram-worker)   | `TELEGRAM_SERVICE`  | Delivery notification |

### Development

```bash
bun test workers/report-worker
```

### Mesh interconnect

| Direction | Peers |
| --------- | ----- |
| **Called by** | Cloudflare Cron (`0 8 * * *`, `0 18 * * *`) and on-demand internal callers / dashboard. |
| **This worker calls** | See list below |

- **[d1-worker](https://github.com/hoox-sh/d1-worker)** — D1_SERVICE — trade history / positions for templates
- **[telegram-worker](https://github.com/hoox-sh/telegram-worker)** — TELEGRAM_SERVICE — report-ready notifications

Full mesh (all isolates live as git submodules under [`hoox-sh/hoox`](https://github.com/hoox-sh/hoox) `workers/`):

| Isolate | Role | Repository |
| ------- | ---- | ---------- |
| [hoox-worker](https://github.com/hoox-sh/hoox-worker) | Public webhook gateway (WAF, idempotency, dispatch) | monorepo `workers/hoox-worker` |
| [trade-worker](https://github.com/hoox-sh/trade-worker) | Multi-exchange order execution (Binance / Bybit / MEXC) | monorepo `workers/trade-worker` |
| [agent-worker](https://github.com/hoox-sh/agent-worker) | AI risk manager (configurable cron 1–1440 min, kill switch) | monorepo `workers/agent-worker` |
| [d1-worker](https://github.com/hoox-sh/d1-worker) | D1 SQL proxy + settings / balances / positions | monorepo `workers/d1-worker` |
| [telegram-worker](https://github.com/hoox-sh/telegram-worker) | Alerts, bot commands, RAG copilot | monorepo `workers/telegram-worker` |
| [email-worker](https://github.com/hoox-sh/email-worker) | Mailgun / email signal parsing → trade | monorepo `workers/email-worker` |
| [analytics-worker](https://github.com/hoox-sh/analytics-worker) | Analytics Engine write + query path | monorepo `workers/analytics-worker` |
| [report-worker](https://github.com/hoox-sh/report-worker) | PDF reports via Browser Rendering → R2 | monorepo `workers/report-worker` |
| [web3-wallet-worker](https://github.com/hoox-sh/web3-wallet-worker) | On-chain wallet identity (ethers.js) | monorepo `workers/web3-wallet-worker` |
| [dashboard](https://github.com/hoox-sh/hoox/tree/main/workers/dashboard) | Next.js ops console (OpenNext, public) | monorepo `workers/dashboard` |

### Docs & monorepo

| Resource | Link |
| -------- | ---- |
| Isolate profile (operators) | [https://docs.hoox.sh/docs/devops/workers/report-worker](https://docs.hoox.sh/docs/devops/workers/report-worker) |
| Parent monorepo | [github.com/hoox-sh/hoox](https://github.com/hoox-sh/hoox) |
| This repository | [github.com/hoox-sh/report-worker](https://github.com/hoox-sh/report-worker) |
| Workers index | [docs.hoox.sh → Workers](https://docs.hoox.sh/docs/devops/workers) |
| CLI | `@hoox-sh/hoox-cli` · `hoox deploy worker report-worker` |

### License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — part of the HOOX open-core mesh.
