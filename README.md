# EcomfyApp Buyer Monitoring System

MVP system that automatically detects buyers with operational risk, classifies them by severity, sends real-time Slack alerts, and generates AI-powered daily executive summaries.

## Tech Stack

- **n8n** — Workflow orchestrator (self-hosted)
- **Google Sheets** — Simulated buyer database + results + audit log
- **Slack** — Real-time alerts via webhook
- **OpenAI API** (GPT-4o-mini) — AI-generated executive summaries

## How It Works

1. Cron trigger fires daily at 8:00 AM EST
2. Reads 20 buyers from Google Sheets
3. Calculates a Health Score (0–100) for each buyer using deterministic logic
4. Classifies buyers: Healthy / Warning / Critical / Needs Review / Inactive
5. Sends individual Slack alerts for at-risk buyers (Critical, Warning, Needs Review)
6. Generates an AI-powered daily executive summary
7. Writes results and execution log back to Google Sheets

## Repository Structure

```
├── data/
│   └── buyers.csv                    # 20 simulated buyers (all required edge cases)
├── workflows/
│   └── buyer-monitoring.json         # n8n workflow export (13 nodes)
├── docs/
│   ├── technical-documentation.md    # Architecture, logic, error handling, production path
│   ├── final-summary.md             # What was completed, limitations, next steps
│   └── 2025-05-20-buyer-monitoring-design.md  # Design spec
├── .env.example                      # Environment variables template
├── .gitignore
└── README.md
```

## Health Score Formula

Starts at 100, deducts points per issue:

| Condition | Deduction |
|-----------|-----------|
| Balance ≤ $0 | -30 |
| Campaign inactive | -25 |
| No ping tree assigned | -25 |
| 0 leads today (active buyer) | -20 |
| Daily cap full | -20 |
| High cancellation risk | -20 |
| Low balance ($0–$50) | -15 |
| No leads in 48h+ | -15 |
| 5+ complaints (7 days) | -15 |
| 10+ rejected leads (7 days) | -15 |
| Contactability < 30% | -15 |
| Auto-recharge failure | -15 |
| Outside schedule | -10 |
| 3–5 complaints (7 days) | -10 |
| Contactability 30–50% | -10 |

**Classification:** 90–100 = Healthy | 70–89 = Warning | 50–69 = Needs Review | 0–49 = Critical

## Links

- **Google Sheet:** [Ecomfy Buyer Monitoring](https://docs.google.com/spreadsheets/d/168r8sEUXPlcbh9wJtbZ9wgN29nxgGY1BBs0yVEbalA0)
- **Technical Documentation:** [docs/technical-documentation.md](docs/technical-documentation.md)
- **Final Summary:** [docs/final-summary.md](docs/final-summary.md)
