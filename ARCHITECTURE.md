# Architecture

This document explains how the workflow is built, what each node does, and how data flows through the system.

## High-Level Flow

```
┌─────────────────────────────────────────────────────────┐
│  Every Monday (Scheduled Trigger)                       │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  CRM (Google Sheets)                 │
│  Reads full sales pipeline           │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Weekly Report Data (Code)           │
│  Filters last 7 days of records      │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Calculate KPI's (Code)              │
│  Totals: wins, losses, revenue,      │
│  top performers                      │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Sales Insight Agent (OpenAI)        │
│  Transforms metrics into narrative   │
│  Generates insights & analysis       │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  End Report (Slack)                  │
│  Posts formatted report to channel   │
└──────────────────────────────────────┘
```

---

## Node-by-Node Breakdown

### 1. **7 day's once Trigger**
- **Type:** Schedule Trigger
- **What it does:** Fires every Monday at 8 AM (configurable)
- **Output:** Starts the workflow

**Configuration:**
```
Rule: Recurring
Interval: weeks (Every 1 week)
```

---

### 2. **CRM** (Google Sheets Node)
- **Type:** Google Sheets API (Read)
- **What it does:** Reads all rows from your sales pipeline spreadsheet
- **Credentials:** Google Sheets OAuth2 API
- **Input:** Spreadsheet ID, Sheet name
- **Output:** Array of all sales records

**Data it retrieves:**
```json
[
  {
    "sales_agent": "Alice Johnson",
    "deal_stage": "Won",
    "close_value": 15000,
    "date_history": "15-01-2024"
  },
  ...
]
```

**Why this step?**
Pulls the raw data. No filtering yet — we get everything from the sheet.

---

### 3. **Weekly Report Data** (JavaScript Code)
- **Type:** Code node (JavaScript v2)
- **What it does:** Filters records to only include the past 7 days
- **Input:** All CRM records from previous node
- **Output:** Only records from the last 7 days

**How it works:**
```javascript
const today = new Date();
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(today.getDate() - 7);

// Keep only rows where date_history falls within the last 7 days
return items.filter(item => {
  const dateStr = item.json.date_history;
  const [day, month, year] = dateStr.split('-').map(Number);
  const rowDate = new Date(year, month - 1, day);
  return rowDate >= sevenDaysAgo && rowDate <= today;
});
```

**Why this step?**
You want weekly insights, not all-time data. This isolates the past 7 days only.

---

### 4. **Calculate KPI's** (JavaScript Code)
- **Type:** Code node (JavaScript v2)
- **What it does:** Analyzes the weekly data and extracts key metrics
- **Input:** Weekly records from previous node
- **Output:** Single object with aggregated KPIs

**Metrics calculated:**
```javascript
{
  totalOpportunities: 45,        // Total rows in the week
  wonDeals: 12,                  // Rows where deal_stage = "Won"
  lostDeals: 5,                  // Rows where deal_stage = "Lost"
  totalRevenue: 180000,          // Sum of close_value for won deals
  topAgents: [
    { agent: "Alice Johnson", wins: 5 },
    { agent: "Bob Smith", wins: 4 },
    { agent: "Carol Davis", wins: 3 }
  ]
}
```

**Why this step?**
Raw data → meaningful numbers. We aggregate and rank.

---

### 5. **OpenAI Chat Model** (Language Model)
- **Type:** LLM (Large Language Model)
- **What it does:** Provides the AI engine for the agent
- **Model:** GPT-4 Mini (cost-effective, fast)
- **Credentials:** OpenAI API key
- **Purpose:** Available to the agent node for reasoning

**Why this step?**
Agents need a language model. This sets it up.

---

### 6. **Sales Insight Agent** (OpenAI Agent)
- **Type:** Agent node (reasoning + function calling)
- **What it does:** Takes the KPIs and generates a professional, conversational report
- **Input:** KPI object from Calculate KPI's node
- **Output:** Markdown-formatted report

**The prompt it uses:**
```
You are a Sales Operations Analyst.
Analyze the metrics and write a weekly Slack update.

Include:
- Executive summary (1-2 sentences, conversational)
- Pipeline Activity section (formatted)
- Top Performers (with emojis 🥇🥈🥉)
- Key Insights (2-3 meaningful observations)

Keep professional, friendly, data-driven tone.
```

**Sample output:**
```
📈 Weekly Sales Update

Great week for the team! Deals are closing strong, 
and our top performers are on fire.

━━━━━━━━━━━━━━

📊 Pipeline Activity
• Opportunities Reviewed: 45
• Deals Won: 12
• Deals Lost: 5
• Revenue Generated: $180,000

━━━━━━━━━━━━━━

🏆 Top Performers
🥇 Alice Johnson — 5 wins
🥈 Bob Smith — 4 wins
🥉 Carol Davis — 3 wins

━━━━━━━━━━━━━━

💡 Key Insights
• Win rate improved 15% vs. last week
• Top 3 agents accounted for 60% of revenue
• Average deal size: $15,000

Have a great week ahead! 🎯
```

**Why this step?**
Numbers alone don't tell the story. AI transforms metrics into intelligence.

---

### 7. **End Report** (Slack)
- **Type:** Slack Send Message node
- **What it does:** Posts the generated report to a Slack channel
- **Credentials:** Slack OAuth2 API (bot token)
- **Input:** The report text from Sales Insight Agent
- **Output:** Message appears in Slack

**Configuration:**
```
Channel ID: C0BBKPXQWQG (your channel)
Message Text: {{ $json.output }}  (the AI-generated report)
Include Link to Workflow: false
```

**Why this step?**
Gets the report in front of the team instantly.

---

## Data Flow Example

**Starting data (Google Sheets):**
```
sales_agent | deal_stage | close_value | date_history
Alice       | Won        | 20000       | 15-01-2024
Bob         | Won        | 15000       | 14-01-2024
Carol       | Lost       | 5000        | 10-01-2024
Alice       | Won        | 25000       | 08-01-2024
```

**After Weekly Report Data filter (last 7 days):**
```
Alice       | Won        | 20000       | 15-01-2024
Bob         | Won        | 15000       | 14-01-2024
```
(Carol's record is older than 7 days, so it's filtered out)

**After Calculate KPI's:**
```json
{
  "totalOpportunities": 2,
  "wonDeals": 2,
  "lostDeals": 0,
  "totalRevenue": 35000,
  "topAgents": [
    { "agent": "Alice", "wins": 1 },
    { "agent": "Bob", "wins": 1 }
  ]
}
```

**After Sales Insight Agent (AI writes):**
```
📈 Weekly Sales Update

Solid start to the week with two wins closing.
Alice and Bob are crushing it!

━━━━━━━━━━━━━━

📊 Pipeline Activity
• Opportunities Reviewed: 2
• Deals Won: 2
• Deals Lost: 0
• Revenue Generated: $35,000

...
```

**Finally (Slack):**
Report posts to #sales-insights channel ✅

---

## Key Design Decisions

### 1. **Why filter to 7 days?**
Most teams think weekly. You can easily change this to 14 days, 30 days, etc.

### 2. **Why use an Agent instead of a simple prompt?**
Agents can reason, decide what matters, and adapt. Agents produce better narratives than static templates.

### 3. **Why separate Calculate KPI's from Report Data?**
Separation of concerns. One node = one job. Easier to debug, modify, or reuse.

### 4. **Why Google Sheets as the source?**
Most sales teams already use it. Easy to share, no database setup needed.

### 5. **Why schedule for Mondays?**
Weekly cadence is standard for sales reviews. You can change to any day/time.

---

## Customization Points

### Change the reporting frequency
Edit the **7 day's once Trigger** node:
- Change from "weeks" to "days" (daily reports)
- Change from "weeks" to "months" (monthly reports)

### Add more KPIs
Edit the **Calculate KPI's** code to include:
- Average deal size: `totalRevenue / wonDeals`
- Win rate: `(wonDeals / totalOpportunities) * 100`
- Pipeline value: Sum of all pending deals

### Change report format
Edit the **Sales Insight Agent** system prompt to:
- Add sections (e.g., "Risk Assessment")
- Change tone (more formal, more casual)
- Include specific metrics

### Add more data sources
Chain additional Google Sheets reads or APIs:
- HubSpot CRM instead of Sheets
- Stripe for revenue data
- Zendesk for customer support metrics

---

## Performance & Costs

### Execution Time
- ~10–15 seconds per run
- Mostly waiting for OpenAI response

### API Costs (per week)
- **Google Sheets:** Free (100 requests/min limit)
- **OpenAI:** ~$0.01–$0.05 (GPT-4 mini, 1 request)
- **Slack:** Free

**Total monthly cost:** ~$0.10–$0.20 (or free with OpenAI free tier)

### Scaling
- Can handle 1000s of CRM records without issue
- If you add more workflows, consider n8n plan limits

---

## Troubleshooting

**Workflow runs but Slack gets no message:**
- Check the **End Report** node has the right channel ID
- Check the bot has permission to post there
- Look at execution logs (click the workflow run in history)

**Data looks wrong:**
- Check **CRM** is reading the right sheet
- Verify column names match exactly (case-sensitive)
- Ensure date format is DD-MM-YYYY

**OpenAI fails:**
- API key invalid or expired
- Out of credits (check platform.openai.com/account/usage)
- Rate limit hit (usually recovers in seconds)

---

## Next Steps

- **Want to learn more n8n?** Check out [n8n documentation](https://docs.n8n.io)
- **Want to customize the report?** Edit the **Sales Insight Agent** prompt
- **Want to add data sources?** Add new nodes between **CRM** and **Weekly Report Data**

