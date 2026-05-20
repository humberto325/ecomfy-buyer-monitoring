# EcomfyApp Buyer Monitoring System - Design Spec

**Date:** May 20, 2025
**Author:** Humberto
**Status:** Approved

---

## 1. Problem Statement

Ecomfy Lead distributes leads to buyers (insurance agents). Sometimes a buyer appears fully active (positive balance, available cap, active schedule, active campaign, assigned ping tree) but receives zero leads. These cases go undetected until the buyer complains or cancels.

**Goal:** Build an MVP that automatically detects buyers with operational risk, classifies them by severity, sends Slack alerts, and generates a daily executive summary.

---

## 2. Architecture Overview

```
┌─────────────┐
│ Cron Trigger │  (daily at 8:00 AM EST)
└──────┬──────┘
       │
┌──────▼──────────┐
│ Google Sheets    │  (read "Buyers" sheet)
└──────┬──────────┘
       │
┌──────▼──────────┐
│ Code Node        │  (Health Score + Status classification)
│ "Score Engine"   │
└──────┬──────────┘
       │
┌──────▼──────────┐
│ Switch Node      │  (route by status)
│                  │
├─► Critical ──────┼─► Slack Alert (red, urgent)
├─► Warning ───────┼─► Slack Alert (yellow)
├─► Needs Review ──┼─► Slack Alert (blue)
├─► Healthy ───────┼─► (no alert)
└─► Inactive ──────┼─► (no alert)
       │
┌──────▼──────────┐
│ Code Node        │  (aggregate stats, top 5 issues)
│ "Build Summary"  │
└──────┬──────────┘
       │
┌──────▼──────────┐
│ OpenAI Node      │  (generate executive summary)
└──────┬──────────┘
       │
┌──────▼──────────┐
│ Slack Node       │  (send daily summary)
└──────┬──────────┘
       │
┌──────▼──────────┐
│ Google Sheets    │  (write to "Results" + "Log")
└─────────────────┘
```

### Tech Stack

| Tool | Role | Why |
|------|------|-----|
| Google Sheets | Simulated buyer database | Ecomfy uses Sheets internally; familiar, visual, easy to review |
| n8n | Workflow orchestrator | Core tool for the role; visual, self-hosted, extensible |
| Slack | Real-time alerts | Industry standard for ops teams; real webhook for demo |
| OpenAI API | AI-powered summaries | Generate natural language executive summaries and action suggestions |

---

## 3. Data Structure (Google Sheets)

### Sheet 1: Buyers (input data)

20 simulated buyers with these columns:

| Column | Type | Description |
|--------|------|-------------|
| buyer_id | number | Unique ID (101-120) |
| buyer_name | string | Fictional name |
| status | string | "active" or "inactive" |
| product | string | Insurance type (Auto, Health, Life, Home, Medicare) |
| balance | number | Current account balance in USD |
| auto_recharge_enabled | boolean | TRUE/FALSE |
| daily_cap | number | Max leads per day |
| leads_received_today | number | Leads received today |
| leads_received_yesterday | number | Leads received yesterday |
| schedule_start | time | Daily schedule start (HH:MM) |
| schedule_end | time | Daily schedule end (HH:MM) |
| timezone | string | Buyer timezone (EST, CST, PST, etc.) |
| states_selected | string | US states targeted |
| campaign_active | boolean | TRUE/FALSE |
| ping_tree_assigned | boolean | TRUE/FALSE |
| last_lead_received_at | datetime | Timestamp of last lead |
| last_recharge_at | datetime | Timestamp of last auto-recharge |
| last_support_message_at | datetime | Last support interaction |
| complaint_count_last_7_days | number | Complaints in past 7 days |
| rejected_leads_last_7_days | number | Rejected leads in past 7 days |
| contactability_rate | number | Percentage (0-100) |
| cancellation_risk | string | "low", "medium", "high" |
| notes | string | Free text notes |

### Distribution of buyer scenarios:

- 6 Healthy buyers (good balance, receiving leads, no issues)
- 4 Warning buyers (some issues: complaints, low contactability, few leads)
- 4 Critical buyers (active but not receiving leads, major issues)
- 3 Inactive buyers (status = inactive)
- 3 Needs Review buyers (auto-recharge issues, missing config)

### Sheet 2: Results (output)

Written by n8n after each run:

| Column | Description |
|--------|-------------|
| buyer_id | Reference to buyer |
| buyer_name | Buyer name |
| health_score | Calculated score (0-100) |
| classification | Healthy / Warning / Critical / Inactive / Needs Review |
| issues_detected | List of issues found |
| suggested_actions | Recommended actions |
| alert_sent | TRUE/FALSE |
| evaluated_at | Timestamp |

### Sheet 3: Log (audit trail)

| Column | Description |
|--------|-------------|
| run_id | Unique execution ID |
| timestamp | When the workflow ran |
| total_buyers | Total buyers evaluated |
| critical_count | Number of critical buyers |
| warning_count | Number of warning buyers |
| healthy_count | Number of healthy buyers |
| needs_review_count | Number needing review |
| inactive_count | Number inactive |
| alerts_sent | Total alerts sent |
| summary_sent | TRUE/FALSE |
| errors | Any errors encountered |

---

## 4. Health Score Logic

### Formula

Start at 100 points. Deduct based on detected issues:

| Condition | Deduction | Rationale |
|-----------|-----------|-----------|
| balance <= 0 | -30 | Cannot buy leads |
| balance > 0 but < 50 | -15 | Low balance risk |
| daily_cap full (received >= cap) | -20 | No room for more leads |
| Outside active schedule | -10 | Expected, minor flag |
| campaign_active = FALSE | -25 | Cannot receive leads |
| ping_tree_assigned = FALSE | -25 | Cannot receive leads |
| 0 leads today (while active) | -20 | Core problem to detect |
| No leads received in 48h+ | -15 | Extended inactivity |
| complaint_count > 5 (7 days) | -15 | High complaint volume |
| complaint_count 3-5 (7 days) | -10 | Moderate complaints |
| rejected_leads > 10 (7 days) | -15 | Quality/config issue |
| contactability_rate < 30% | -15 | Poor lead utilization |
| contactability_rate 30-50% | -10 | Below average utilization |
| cancellation_risk = "high" | -20 | Churn risk |
| auto_recharge ON + balance < 50 + no recharge in 48h | -15 | Billing system issue |

**Minimum score: 0** (does not go negative)

### Classification

| Score Range | Status |
|-------------|--------|
| 90-100 | Healthy |
| 70-89 | Warning |
| 50-69 | Needs Review |
| 0-49 | Critical |
| status = "inactive" | Inactive (overrides score) |

---

## 5. Slack Alert Formats

### Critical Alert (Red)

```
🔴 CRITICAL - Buyer Monitoring Alert
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Buyer: {buyer_name} (ID: {buyer_id})
Status: Critical | Score: {score}/100
Product: {product}

Reason: {issues_detected}

Suggested Action:
• {action_1}
• {action_2}
• {action_3}

Priority: 🔴 High
Owner: Operations / Tech
Detected at: {timestamp}
```

### Warning Alert (Yellow)

```
🟡 WARNING - Buyer Monitoring Alert
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Buyer: {buyer_name} (ID: {buyer_id})
Score: {score}/100
Reason: {issues_detected}

Suggested Action:
• {action_1}
• {action_2}
```

### Needs Review Alert (Blue)

```
🔵 NEEDS REVIEW - Buyer Monitoring Alert
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Buyer: {buyer_name} (ID: {buyer_id})
Score: {score}/100
Reason: {issues_detected}

Suggested Action:
• {action_1}
```

### Daily Executive Summary

```
📊 Daily Buyer Monitoring Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Date: {date} | Reviewed: {total} buyers

✅ Healthy: {n}    🟡 Warning: {n}
🔴 Critical: {n}   ⚪ Inactive: {n}
🔵 Needs Review: {n}

⚠️ Top 5 Urgent Issues:
1. {issue}
2. {issue}
3. {issue}
4. {issue}
5. {issue}

💡 Suggested Actions:
{AI-generated natural language summary}

💰 Estimated Revenue Risk: ${amount}/day
📝 Operational Notes: {notes}
```

---

## 6. AI Usage (OpenAI)

### Where AI is used:

1. **Daily Executive Summary** — Takes aggregated stats + top issues and generates a professional, human-readable summary for stakeholders.

2. **Action Suggestions** — Given a critical buyer's context, AI suggests specific areas to investigate (Phonexa filters, CRM config, ping tree, etc.)

### Where AI is NOT used:

- Health Score calculation (deterministic logic)
- Buyer classification (rule-based)
- Data processing (code nodes)

### Prompt structure:

```
You are an operations analyst for a lead distribution company.
Given the following buyer monitoring data, generate a concise
executive summary for the operations team.

Data: {aggregated_stats}
Top issues: {top_5_issues}

Include: overview, urgent items, suggested actions, revenue risk estimate.
Tone: professional, direct, actionable.
```

---

## 7. Error Handling

| Scenario | Strategy |
|----------|----------|
| Google Sheets unavailable | Retry 3 times (30s intervals). If fails, send Slack alert: "Data source unavailable" |
| Slack webhook fails | Log error, continue. Results still saved to Sheets |
| OpenAI API fails | Send summary without AI (raw data only). Don't block the workflow |
| Duplicate alerts | Check "Log" sheet for today's entries before sending. Skip if already sent |
| Invalid buyer data | Skip buyer, log warning, continue processing others |
| Workflow crash | n8n error workflow sends notification to Slack |

---

## 8. Scalability Notes (for documentation)

### Current MVP limitations:
- Google Sheets as data source (max ~1000 rows practical)
- Single daily run (not real-time)
- No historical trend analysis
- No user authentication on results

### Production evolution path:
- Replace Sheets with PostgreSQL/Supabase for real data
- Connect to Phonexa API, GHL API, real CRM
- Add real-time monitoring (every 15-30 min)
- Add historical dashboards (trends, patterns)
- Add webhook endpoints for real-time data ingestion
- Add role-based access for ops team
- Integrate with PagerDuty/OpsGenie for critical alerts

---

## 9. Deliverables Checklist

- [ ] Google Sheet with 20 buyers + Results + Log sheets
- [ ] n8n workflow (exported JSON + live demo)
- [ ] Real Slack alerts in #buyer-alerts channel
- [ ] Technical documentation (architecture, logic, tools, limitations, next steps)
- [ ] Loom video (max 7 min)
- [ ] Final summary document
- [ ] GitHub repo with all code and docs
