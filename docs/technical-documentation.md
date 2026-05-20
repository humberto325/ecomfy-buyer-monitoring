# EcomfyApp Buyer Monitoring System — Technical Documentation

## 1. Architecture Overview

### System Diagram

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Google       │     │     n8n           │     │    Slack         │
│  Sheets       │◄───►│  Workflow Engine   │────►│  #buyer-alerts   │
│  (Data Layer) │     │  (Orchestrator)   │     │  (Notifications) │
└──────────────┘     └────────┬──────────┘     └─────────────────┘
                              │
                     ┌────────▼──────────┐
                     │   OpenAI API       │
                     │   (AI Summaries)   │
                     └───────────────────┘
```

### How It Works

1. **Cron Trigger** fires daily at 8:00 AM EST
2. **n8n reads** all buyer data from Google Sheets ("Buyers" tab)
3. **Score Engine** calculates a Health Score (0–100) for each buyer and assigns a classification (Healthy / Warning / Critical / Needs Review / Inactive)
4. The workflow splits into two parallel branches:
   - **Alert Branch**: filters non-healthy buyers, formats Slack alerts, and sends them individually
   - **Summary Branch**: aggregates stats, calls OpenAI to generate an executive summary, and sends it to Slack
5. Results and execution logs are written back to Google Sheets

### Tech Stack

| Tool | Role | Why This Tool |
|------|------|---------------|
| **Google Sheets** | Simulated buyer database + results storage | Ecomfy uses Sheets internally; visual, easy to review, requires no infra setup |
| **n8n** | Workflow orchestrator | Core tool for the role; self-hosted, visual, supports Code nodes for custom logic |
| **Slack** | Real-time alerts via webhook | Industry standard for ops teams; real-time delivery; rich message formatting |
| **OpenAI API** (gpt-4o-mini) | AI-powered executive summaries | Generates natural language summaries and action recommendations; cost-effective model |

---

## 2. Data Structure

### Google Sheet: "Ecomfy Buyer Monitoring"

#### Sheet 1: Buyers (Input)

20 simulated buyers with 23 columns covering all required fields:

`buyer_id`, `buyer_name`, `status`, `product`, `balance`, `auto_recharge_enabled`, `daily_cap`, `leads_received_today`, `leads_received_yesterday`, `schedule_start`, `schedule_end`, `timezone`, `states_selected`, `campaign_active`, `ping_tree_assigned`, `last_lead_received_at`, `last_recharge_at`, `last_support_message_at`, `complaint_count_last_7_days`, `rejected_leads_last_7_days`, `contactability_rate`, `cancellation_risk`, `notes`

**Buyer scenario distribution:**

| Scenario | Count | Buyer IDs |
|----------|-------|-----------|
| Healthy | 4 | 102, 103, 105, 118 |
| Warning | 4 | 106, 108, 119, 120 |
| Critical | 5 | 107, 109, 110, 116, 117 |
| Needs Review | 4 | 101, 104, 111, 112 |
| Inactive | 3 | 113, 114, 115 |

**Edge cases covered:**
- Active buyer with balance + cap + schedule + campaign + ping tree but 0 leads (101, 104)
- Zero balance with inactive campaign (110)
- Daily cap completely full (120)
- No ping tree assigned (111)
- Campaign inactive without buyer awareness (112)
- High complaint count + low contactability + churn risk (109)
- Auto-recharge enabled but not triggered for days (107, 116, 117)

#### Sheet 2: Results (Output)

Written automatically after each run:
`buyer_id`, `buyer_name`, `health_score`, `classification`, `issues_detected`, `suggested_actions`, `alert_sent`, `evaluated_at`

#### Sheet 3: Log (Audit Trail)

One row per execution:
`run_id`, `timestamp`, `total_buyers`, `critical_count`, `warning_count`, `healthy_count`, `needs_review_count`, `inactive_count`, `alerts_sent`, `summary_sent`, `errors`

---

## 3. Health Score Logic

### Formula

Every buyer starts at **100 points**. Points are deducted based on detected issues:

| Condition | Deduction | Rationale |
|-----------|-----------|-----------|
| Balance ≤ $0 | -30 | Cannot purchase leads at all |
| Balance $0–$50 | -15 | At risk of running out |
| Daily cap full (received ≥ cap) | -20 | No room for more leads today |
| Outside active schedule | -10 | Expected, minor operational flag |
| Campaign inactive | -25 | Blocks lead delivery entirely |
| No ping tree assigned | -25 | Blocks lead delivery entirely |
| 0 leads today (active buyer) | -20 | Core problem this system detects |
| No leads in 48+ hours | -15 | Extended inactivity signal |
| 5+ complaints in 7 days | -15 | High dissatisfaction |
| 3–5 complaints in 7 days | -10 | Moderate dissatisfaction |
| 10+ rejected leads in 7 days | -15 | Potential quality/config issue |
| Contactability < 30% | -15 | Very poor lead utilization |
| Contactability 30–50% | -10 | Below average utilization |
| Cancellation risk = "high" | -20 | Imminent churn risk |
| Auto-recharge ON + balance < $50 + no recharge in 48h | -15 | Billing system failure |

**Score floor: 0** (never goes negative)

### Classification Mapping

| Score | Classification | Action Level |
|-------|---------------|--------------|
| 90–100 | Healthy | No action needed |
| 70–89 | Warning | Monitor closely |
| 50–69 | Needs Review | Investigate within 24h |
| 0–49 | Critical | Immediate action required |
| status = "inactive" | Inactive | Overrides score; no action unless reactivation planned |

### Design Rationale

- **Highest deductions (-25, -30)** are for conditions that completely block lead delivery (no balance, no campaign, no ping tree)
- **Medium deductions (-15, -20)** are for conditions that indicate operational problems (zero leads, complaints, billing issues)
- **Lower deductions (-10)** are for expected or temporary conditions (schedule, moderate complaints)
- A buyer with 2–3 major issues falls into Critical, which is the correct alert level
- The formula is **deterministic and auditable** — no AI black box in the scoring

---

## 4. Workflow Nodes (n8n)

### Node Breakdown

| # | Node Name | Type | Purpose |
|---|-----------|------|---------|
| 1 | Daily 8AM Trigger | Schedule Trigger | Fires the workflow daily at 8:00 AM |
| 2 | Read Buyers | Google Sheets | Reads all rows from "Buyers" sheet |
| 3 | Score Engine | Code (JS) | Calculates health_score + classification for each buyer |
| 4 | Filter & Format Alerts | Code (JS) | Filters non-healthy buyers, formats Slack alert payloads |
| 5 | Slack Individual Alerts | HTTP Request | Sends each alert to Slack webhook |
| 6 | Build Summary | Code (JS) | Aggregates stats, top 5 issues, builds OpenAI prompt |
| 7 | OpenAI Summary | HTTP Request | Calls GPT-4o-mini to generate executive summary |
| 8 | Format Summary | Code (JS) | Formats the daily summary as a Slack Block Kit message |
| 9 | Slack Daily Summary | HTTP Request | Sends the summary to Slack |
| 10 | Prepare Results | Code (JS) | Formats scored buyer data for Google Sheets |
| 11 | Write Results | Google Sheets | Appends results to "Results" sheet |
| 12 | Prepare Log | Code (JS) | Builds the execution log entry |
| 13 | Write Log | Google Sheets | Appends log entry to "Log" sheet |

### Parallel Branches

From the Score Engine, the workflow splits into two parallel branches:

- **Alert Branch** (nodes 4–5): sends individual Slack alerts for Critical, Warning, and Needs Review buyers
- **Summary Branch** (nodes 6–13): generates AI summary, sends to Slack, writes results and logs

This ensures alerts are sent immediately without waiting for the AI summary to generate.

---

## 5. Slack Alert Formats

### Individual Alerts

Each non-healthy buyer gets an individual alert with:
- **Color-coded header**: 🔴 Critical / 🟡 Warning / 🔵 Needs Review
- **Buyer details**: name, ID, score, product
- **Issues detected**: bulleted list of specific problems
- **Suggested actions**: bulleted list of recommended steps
- **Metadata**: priority level, owner team, timestamp

### Daily Executive Summary

Sent once per execution, includes:
- **Stats dashboard**: count by classification
- **Top 5 urgent issues**: ordered by severity
- **AI-generated analysis**: natural language summary by OpenAI
- **Revenue risk estimate**: based on daily cap × avg lead price for critical buyers

---

## 6. AI Usage

### Where AI Is Used

1. **Executive Summary Generation**: OpenAI receives aggregated stats + top 5 issues and generates a professional summary for stakeholders
2. **Action Recommendations**: AI suggests specific areas to investigate based on buyer context

### Where AI Is NOT Used

- Health Score calculation → deterministic JavaScript logic
- Buyer classification → rule-based thresholds
- Data reading/writing → native n8n nodes
- Alert formatting → template-based Code nodes

### Why This Separation

The test instructions state: *"AI no debe reemplazar la lógica básica. La lógica operativa debe ser clara y verificable."*

Our scoring and classification logic is 100% deterministic and auditable. AI adds value only in the summary layer — turning structured data into human-readable insights. If OpenAI fails, the system still functions: alerts are sent, results are written, only the summary text falls back to raw data.

### Model Choice

**GPT-4o-mini** — chosen for:
- Cost efficiency (~$0.001 per summary)
- Fast response time (<3 seconds)
- Sufficient quality for structured summarization
- Reliable JSON output

---

## 7. Error Handling

| Scenario | Strategy | Impact |
|----------|----------|--------|
| Google Sheets unavailable | Workflow fails, n8n logs the error | No data processed; retry on next scheduled run |
| Slack webhook fails | HTTP Request returns error, workflow continues | Alerts not delivered but results still written to Sheets |
| OpenAI API fails | Code node catches error, outputs fallback text: "AI summary unavailable" | Summary sent without AI text; all other functions work |
| Invalid buyer data | Score Engine skips buyers with missing critical fields | Partial results; logged in execution |
| Duplicate runs | Log sheet records each execution; can detect double-runs | Manual review if duplicates detected |
| Workflow crash | n8n execution history shows failed run with error details | Ops team can review and re-trigger manually |

### Error Handling in Code

The Format Summary node includes a try/catch for the OpenAI response:
```javascript
try {
  aiSummary = openaiResponse.choices[0].message.content;
} catch (e) {
  aiSummary = 'AI summary unavailable. Please review the data manually.';
}
```

---

## 8. Limitations (Current MVP)

- **Google Sheets as data source**: practical limit ~1,000 rows; not suitable for real-time querying
- **Single daily execution**: issues that arise mid-day aren't detected until next morning
- **No historical trending**: each run is independent; no week-over-week analysis
- **Simulated data**: not connected to real Phonexa, GHL, or CRM systems
- **No duplicate alert prevention**: if run twice in a day, alerts are sent twice
- **Schedule check uses UTC**: timezone conversion is approximate

---

## 9. Production Evolution Path

### Phase 1: Connect Real Data Sources
- Replace Google Sheets with PostgreSQL/Supabase
- Connect to Phonexa API for real lead data
- Connect to Go High Level API for CRM data
- Pull billing data from payment system

### Phase 2: Real-Time Monitoring
- Change trigger from daily to every 15–30 minutes
- Add webhook endpoints for real-time event ingestion
- Implement deduplication logic (alert cooldown per buyer)
- Add alert escalation (if Critical persists > 2 hours, escalate)

### Phase 3: Intelligence Layer
- Historical trend analysis (week-over-week comparisons)
- Predictive churn scoring using ML
- Anomaly detection (sudden drops in lead volume)
- Auto-remediation (auto-pause campaigns that waste budget)

### Phase 4: Team Dashboard
- Web-based dashboard with real-time buyer status
- Role-based access (ops, account managers, leadership)
- Alert management (acknowledge, snooze, escalate)
- Integration with PagerDuty/OpsGenie for on-call rotation

---

## 10. Tools & Technologies Summary

| Category | Tool | Version/Detail |
|----------|------|----------------|
| Orchestration | n8n | Self-hosted (EasyPanel) |
| Data Layer | Google Sheets | Buyer database + results + logs |
| Notifications | Slack | Incoming Webhook |
| AI | OpenAI API | GPT-4o-mini, temperature 0.4 |
| Code | JavaScript | ES6+, within n8n Code nodes |
| Version Control | GitHub | Private repo |
| Hosting | EasyPanel | n8n instance |
