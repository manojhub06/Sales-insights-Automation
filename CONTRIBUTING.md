# Contributing

Want to improve, extend, or customize this workflow? This guide explains how to do it safely and clearly.

## Before You Start

- Read [ARCHITECTURE.md](./ARCHITECTURE.md) to understand how the workflow is built
- This is a personal portfolio project — contributions are for learning and sharing, not production support
- Changes should be tested in a development/staging n8n instance first

---

## Common Modifications

### 1. Change the Report Schedule

**Goal:** Run reports on a different day or more/less frequently

**Steps:**
1. Open the workflow in n8n
2. Click the **7 day's once Trigger** node
3. Edit the rule:
   - **Every week** → Monday morning
   - **Every day** → Daily at 8 AM
   - **Every month** → First day of month
4. Save and test with Execute Workflow

**Example:** Daily reports
- Change interval to "days"
- Set to run at 9 AM

---

### 2. Customize the Report Format

**Goal:** Change sections, tone, or content of the Slack message

**Steps:**
1. Click the **Sales Insight Agent** node
2. Find the **System Message** field
3. Edit the prompt (example below)
4. Save and Execute Workflow to test

**Example: Add a "Risk Assessment" section**

Replace this section in the prompt:
```
💡 Key Insights
• Provide 2-3 meaningful observations based on the data.
```

With this:
```
💡 Key Insights
• Provide 2-3 meaningful observations based on the data.

⚠️ Risk Assessment
• Highlight any concerning trends (e.g., declining win rate)
• Flag deals at risk of falling through
```

**Example: More formal tone**

Add this to the system message:
```
Tone: Professional and formal, suitable for executive review.
Avoid emojis and casual language.
```

---

### 3. Add More KPIs

**Goal:** Track additional metrics like conversion rate, average deal size, etc.

**Steps:**
1. Click the **Calculate KPI's** node
2. Add calculations before the `return` statement:

```javascript
// Add this before the final return
const conversionRate = totalOpportunities > 0 
  ? ((wonDeals / totalOpportunities) * 100).toFixed(1) 
  : 0;

const avgDealSize = wonDeals > 0 ? totalRevenue / wonDeals : 0;

return [
  {
    json: {
      totalOpportunities,
      wonDeals,
      lostDeals,
      totalRevenue,
      conversionRate,          // NEW
      avgDealSize,              // NEW
      topAgents
    }
  }
];
```

3. Update the **Sales Insight Agent** prompt to include these new metrics:

```
📊 Pipeline Activity
• Opportunities Reviewed: X
• Deals Won: X ({{ conversionRate }}% conversion rate)
• Deals Lost: X
• Revenue Generated: X
• Average Deal Size: $X
```

4. Test with Execute Workflow

---

### 4. Connect a Different Data Source

**Goal:** Use HubSpot, Salesforce, or another CRM instead of Google Sheets

**Steps:**
1. Delete the **CRM** node
2. Add a new node for your CRM (e.g., HubSpot, Salesforce)
3. Configure credentials and query
4. Make sure it outputs an array with at least these fields:
   - `sales_agent` or similar (who made the deal)
   - `deal_stage` (Won/Lost/etc.)
   - `close_value` (deal amount)
   - `date_history` (date in any format)
5. Connect the new node to **Weekly Report Data**
6. Adjust the date parsing in **Weekly Report Data** if date format differs

**Example: Connect to HubSpot**
- Add **HubSpot** node
- Query deals
- Map fields:
  - `owner.name` → `sales_agent`
  - `dealstage` → `deal_stage`
  - `amount` → `close_value`
  - `closedate` → `date_history`

---

### 5. Post to Multiple Channels

**Goal:** Send reports to both #sales-insights and #executive-summary

**Steps:**
1. Click the **End Report** node
2. Duplicate it (right-click → Duplicate)
3. Name the new node "End Report - Executive"
4. Change the channel ID to your executive channel
5. (Optional) Customize the message text for that audience
6. Connect both nodes to the **Sales Insight Agent** output

**Result:** Same report posts to two channels automatically.

---

### 6. Add a Filter or Alert

**Goal:** Only post reports when certain conditions are met (e.g., revenue > $100k)

**Steps:**
1. Add an **If** node after **Calculate KPI's**
2. Set condition: `totalRevenue > 100000`
3. Connect true path to **Sales Insight Agent**
4. Connect false path to a **Send Slack Message** node saying "No significant activity"

**Example condition:**
```
Field: totalRevenue
Operator: > (greater than)
Value: 100000
```

---

### 7. Add Data Persistence (Store Reports)

**Goal:** Keep a history of all reports in Google Sheets

**Steps:**
1. Add a **Google Sheets** node (Append Row)
2. Point to a **Report History** sheet with columns:
   - Report_Date
   - Total_Opportunities
   - Won_Deals
   - Revenue
   - Report_Text
3. Map values from **Sales Insight Agent** output
4. Place this node after the Slack post (so it runs after report is sent)

**Why do this?**
- Archive reports for later reference
- Track performance over weeks/months
- Build a dataset for trend analysis

---

## Testing Your Changes

### Test Without Running the Schedule
1. Edit your workflow
2. Click **Execute Workflow** (play button)
3. The workflow runs immediately with current data
4. Check the output in the sidebar
5. If Slack is connected, you'll see a test message post

### Test Specific Nodes
1. Click a node
2. Right-click → **Execute Node**
3. See its output without running the whole workflow

### Check Logs
1. After a workflow executes, click the run in the **Execution** history
2. Expand each node to see input/output
3. Look for error messages

---

## Git Workflow (Optional)

If you're storing this in version control:

```bash
# Clone the repo
git clone <repo-url>
cd sales-pipeline-automation

# Create a branch for your changes
git checkout -b feature/add-conversion-rate

# Make changes, test, commit
git add .
git commit -m "Add conversion rate KPI"

# Push and create a pull request
git push origin feature/add-conversion-rate
```

---

## Best Practices

### ✅ Do
- Test changes in n8n before committing
- Document custom KPIs or report sections
- Keep the workflow modular (one node = one job)
- Use meaningful node names
- Add comments in code nodes if logic is complex

### ❌ Don't
- Hardcode API keys or secrets in the workflow
- Post test messages to production Slack channels
- Remove nodes without understanding what they do
- Change credentials in shared workflows
- Merge branches without testing

---

## Adding Documentation

If you make significant changes, update the relevant docs:

**Changing report format?** → Update ARCHITECTURE.md section "Sales Insight Agent"

**Adding new KPIs?** → Update SETUP.md "Customization Tips"

**Adding a new data source?** → Update ARCHITECTURE.md "Customization Points"

---

## Questions or Issues?

- Check [ARCHITECTURE.md](./ARCHITECTURE.md) for how something works
- Check [SETUP.md](./SETUP.md) for configuration help
- Test in n8n with the Execute button
- Review the execution logs (click the run history)

---

## Sharing Your Improvements

Have a cool customization? Consider:
- Opening an issue to discuss
- Documenting it in a comment for others
- Sharing how you modified it (helps the portfolio!)

This workflow is meant to be learned from and improved. Feel free to make it your own! 🚀

