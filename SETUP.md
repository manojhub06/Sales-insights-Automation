# Setup Guide

This guide walks you through setting up each credential and preparing your data. **No prior API experience needed** — each step is broken down simply.

## Overview

You need to connect three services to n8n:
1. **Google Sheets** (your CRM data)
2. **OpenAI** (AI insights)
3. **Slack** (where reports post)

---

## Step 1: Google Sheets Setup

### Create or Prepare Your Spreadsheet

Your Google Sheets file needs these columns. **Column names must match exactly:**

| Column Name | Type | Example | Purpose |
|---|---|---|---|
| `sales_agent` | Text | "Alice Johnson" | Who made the deal |
| `deal_stage` | Text | "Won", "Lost", "Pending" | Current status |
| `close_value` | Number | 15000 | Deal amount in dollars |
| `date_history` | Text | "15-01-2024" | Date in DD-MM-YYYY format |

**Note:** The workflow filters for "Won" and "Lost" deals. Other stages are counted but not included in revenue.

### Get Your Spreadsheet ID

1. Open your Google Sheets file
2. Copy the ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit
   ```
3. Save this ID — you'll need it

### Create OAuth2 Credentials in n8n

1. In n8n, go to **Credentials** → **New**
2. Search for **"Google Sheets OAuth2 API"**
3. Click **Create**
4. Name it (e.g., "Sales Pipeline Sheets")
5. Click **Google sign-in** button
6. Sign in with your Google account that owns the spreadsheet
7. **Grant permissions** when prompted
8. **Save**

That's it. n8n now has permission to read your sheet.

---

## Step 2: OpenAI Setup

### Get Your API Key

1. Go to [platform.openai.com](https://platform.openai.com)
2. Sign in or create an account
3. Click your profile → **API keys** (or go directly to `/account/api-keys`)
4. Click **Create new secret key**
5. **Copy it immediately** — you won't see it again
6. Keep it safe (don't share it publicly)

### Add to n8n

1. In n8n, go to **Credentials** → **New**
2. Search for **"OpenAI API"**
3. Click **Create**
4. Name it (e.g., "OpenAI Free")
5. Paste your API key in the **API Key** field
6. **Save**

**Cost note:** OpenAI charges per request. With weekly reports using GPT-4 mini, costs are typically $0.01–$0.05 per week. Free trial credits work here.

---

## Step 3: Slack Setup

### Create a Slack Bot

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. Name it: "Sales Insights Bot"
4. Pick your workspace
5. Click **Create App**

### Give It Permissions

1. Left sidebar → **OAuth & Permissions**
2. Scroll to **Scopes** → **Bot Token Scopes**
3. Click **Add an OAuth Scope**
4. Add these scopes:
   - `chat:write` (send messages)
   - `channels:read` (see channels)

### Get Your Bot Token

1. Still in **OAuth & Permissions**, scroll to **OAuth Tokens for Your Workspace**
2. Copy the **Bot User OAuth Token** (starts with `xoxb-`)
3. Keep this safe — don't share it

### Add to n8n

1. In n8n, go to **Credentials** → **New**
2. Search for **"Slack OAuth2 API"**
3. Click **Create**
4. Name it (e.g., "Sales Insights Bot")
5. Paste your **Bot Token** in the **Bot User OAuth Token** field
6. **Save**

### Find Your Channel ID

1. In Slack, right-click the channel where reports should go (e.g., #sales-insights)
2. Copy the **Channel ID** from the menu
3. Or go to the channel → click the name → scroll down → copy ID

---

## Step 4: Import the Workflow

1. In n8n, go to **Workflows** → **Import**
2. Click **Select File** and choose `workflow.json`
3. Click **Import**

The workflow appears but shows **missing credentials** (red exclamation marks).

### Wire Up Credentials

1. Open the workflow
2. For each node with a red exclamation mark:
   - Click the node
   - Under **Credentials**, select the one you just created
   - Save the node

**Nodes to update:**
- **CRM** → Google Sheets OAuth2 API
- **OpenAI Chat Model** → OpenAI API
- **End Report** → Slack OAuth2 API

### Configure the Slack Channel

1. Click the **End Report** node
2. Under **Channel ID**, paste your channel ID from Step 3
3. **Save**

---

## Step 5: Test It

1. Click **Execute Workflow** (play button, top right)
2. The workflow runs immediately (doesn't wait for Monday)
3. Check your Slack channel — you should see the report
4. If it works, move to Step 6

**Troubleshooting:**
- **No Google Sheets data?** Check that your column names match exactly (case-sensitive)
- **Slack error?** Verify your channel ID and bot has access to that channel
- **OpenAI error?** Check your API key is valid and has available credits

---

## Step 6: Activate & Schedule

1. In the workflow, find the **7 day's once Trigger** node
2. It's already set to **Every week** (run Mondays)
3. Click the workflow name at the top → **Save**
4. Toggle the **Active** switch to **ON** (top right, near Execute)

✅ **Done!** The workflow now runs every Monday automatically.

---

## Customization Tips

### Change the Report Day
In the **7 day's once Trigger** node, click the rule settings and pick a different day.

### Change the Lookback Period
In the **Weekly Report data** node, find this line:
```javascript
sevenDaysAgo.setDate(today.getDate() - 7);
```
Change `7` to any number (e.g., `14` for 2 weeks, `30` for 1 month).

### Edit the Slack Channel
Click **End Report** node → update **Channel ID** to any other channel.

### Customize the Report Format
Click **Sales Insight Agent** node → edit the system message to change tone, sections, or content.

---

## Troubleshooting

**"Authentication failed"**
- Credentials may have expired. Redo that credential in n8n.

**"No data found in spreadsheet"**
- Column names don't match. Double-check the spelling and capitalization.
- Spreadsheet is empty. Add at least one row of data.

**"Slack channel not found"**
- Wrong channel ID. Copy it again from Slack.
- Bot doesn't have access. Make sure the bot is invited to the channel.

**"OpenAI API error"**
- API key is invalid or has no credits.
- Check your OpenAI account at platform.openai.com/account/usage.

---

Need help? Check the [ARCHITECTURE.md](./ARCHITECTURE.md) for a deeper dive into how everything connects.
