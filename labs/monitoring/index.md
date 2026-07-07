# Know What It's Doing – Monitoring and Service Health

## The Scenario

You have built a lot across two days: resource groups, VMs, a network, storage, a database, identities, and governance rules. Everything your lead asked for exists. Which is exactly why they now ask the question that separates a demo from a production system:

> "Great — it's all running. How would we *know* if it stopped? Right now, if a resource got deleted or a service went sideways, we'd find out when a customer complained. I want to see what's happening across everything we've built, I want to query it, and I want to be told — automatically — when something important happens instead of stumbling onto it. And when Azure itself has a bad day, I want to know it's them, not us."

That is **observability**, and it is the discipline that makes everything else trustworthy: without it, all your careful building is running blind. In this lab you will meet Azure's monitoring stack from the ground up. You will create a **Log Analytics workspace**, route the **Activity Log** — the same audit trail you first met in Day 1 — into it, and query that data with **KQL**, Azure's query language. You will wire up an **alert** with an **action group** so a significant event emails you unprompted. And you will tour **Service Health**, **Resource Health**, and **Azure Advisor** — the pages that tell you whether a problem is yours or Microsoft's.

This lab is the natural capstone: everything you built, you now learn to watch.

> **Cost note:** A Log Analytics workspace has a generous free ingestion allowance; this lab stays well within it. Action groups and Activity Log alerts are free. Everything is deleted in cleanup.

### Prerequisites

- Completion of Day 1 (you will query the Activity Log those labs generated), or comfort with the portal and Cloud Shell
- An Azure account (the free account is enough)

---

## Step 1 – Recall Where This Started: the Activity Log

Before building anything, look at what you already have. In the portal, search for **Activity log** (or open any resource group and its **Activity log** blade). Scroll back through the last two days.

You are looking at a record of nearly everything you did in this course: resource groups created, a VM started, a lock applied, a policy assignment made, a storage account denied and then allowed.

**Why begin with a page you already met in Day 1?** Because monitoring is not a new system bolted on at the end — it is a *lens* onto signals Azure was recording all along. The Activity Log is the **control-plane audit trail**: every create, update, and delete that went through Azure Resource Manager, from any tool, is here. In Day 1 you used it to prove the portal and CLI share one control plane. Now you will treat it as a *data source* — route it somewhere queryable, ask it questions, and alert on it. The signal was always there; this lab is about *using* it.

**What are the other kinds of signal?** Azure Monitor collects three broad types: **metrics** (numeric, time-series — CPU %, request count, sampled every minute), **logs** (structured event records — the Activity Log, resource diagnostics, application traces), and **traces** (the path of a request through an app, via Application Insights). Metrics answer "how much, right now?"; logs answer "what happened, and when?"; traces answer "why is this specific request slow?" You will work with logs today because that is where the Activity Log lives.

---

## Step 2 – Create a Log Analytics Workspace

The Activity Log you just viewed is not, by itself, richly queryable — you filter it, but you cannot *join* or *aggregate* it. To do that you send it to a **Log Analytics workspace**. Create one:

```bash
az group create \
  --name rg-monitor-<yourinitials> \
  --location eastus \
  --tags project=catalog environment=dev owner=<yourinitials>

az monitor log-analytics workspace create \
  --resource-group rg-monitor-<yourinitials> \
  --workspace-name log-catalog-<yourinitials> \
  --location eastus
```

**What is a Log Analytics workspace, and why is it the center of everything?** It is a queryable store for log and metric data, and it is the engine behind most of Azure Monitor. Diagnostic data from dozens of sources — the Activity Log, VM performance counters, Key Vault access logs, application telemetry — can all flow into one workspace, where a single query language reaches across all of it. That consolidation is the whole value: instead of ten separate log views you cannot correlate, you get one place to ask "show me every delete, from every resource, in the last hour." Think of it as the observability counterpart to what Azure Resource Manager is for control: one hub, many feeds.

---

## Step 3 – Route the Activity Log into the Workspace

The workspace is empty. Connect the subscription's Activity Log to it with a **diagnostic setting**:

```bash
SUB_ID=$(az account show --query id -o tsv)
WS_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-monitor-<yourinitials> \
  --workspace-name log-catalog-<yourinitials> --query id -o tsv)

az monitor diagnostic-settings subscription create \
  --name activity-to-la \
  --location eastus \
  --workspace "$WS_ID" \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Policy","enabled":true},{"category":"Security","enabled":true}]'
```

**What is a diagnostic setting, and why isn't the data just *there*?** Because Azure does not pipe every signal everywhere by default — you would drown in data and pay to store it. A **diagnostic setting** is an explicit routing rule: "send *these* log categories from *this* source to *that* destination." You just told the subscription's Activity Log to send its Administrative, Policy, and Security categories to your workspace. Destinations can also be a storage account (cheap long-term retention) or an event hub (streaming to external tools) — you chose the workspace because you want to *query*. This explicit-routing model is the same discipline as everything else in Azure: nothing happens to your data until you say so.

**Why does the data take a while to appear?** Ingestion has latency — events are collected, batched, and indexed before they are queryable. For a brand-new workspace the *first* Activity Log records can take **15–30+ minutes** to land (later data arrives faster, within a few minutes). Until that first batch arrives, the `AzureActivity` table does not exist yet, so a query against it errors with "table/entity not found" rather than returning an empty result — that error *is* the latency, not a typo. This delay is worth internalizing: monitoring shows you the recent past, not the live instant, which is exactly why alerting (Step 6) matters — you cannot sit watching a query. Start the timer now; you will run your first KQL query after the next couple of steps, and if the table is still missing then, keep waiting and retry.

---

## Step 4 – Generate a Signal Worth Finding

While ingestion catches up, create a small, deliberate event you can later hunt for in the logs — create and delete a resource group:

```bash
az group create --name rg-monitor-signal-<yourinitials> --location eastus \
  --tags project=catalog environment=dev owner=<yourinitials>

az group delete --name rg-monitor-signal-<yourinitials> --yes
```

**Why manufacture an event on purpose?** Because a monitoring lab needs something recent and known to find — and because a *delete* is exactly the kind of event your lead wants to catch ("if a resource got deleted... we'd find out when a customer complained"). You now know a resource-group creation and deletion happened, by you, in the last minute. In Step 6 you will build an alert that would have emailed you about it automatically. Deliberately generating a signal, then confirming your monitoring catches it, is how you *test* observability rather than assume it.

---

## Step 5 – Query the Activity Log with KQL

Give ingestion 5–15 minutes from Step 3, then query. In the portal, open your workspace **log-catalog-<yourinitials>** → **Logs** (dismiss the query-samples popup). Run:

```kusto
AzureActivity
| where TimeGenerated > ago(1d)
| summarize count() by OperationNameValue, ActivityStatusValue
| order by count_ desc
```

Then find your deliberate delete from Step 4:

```kusto
AzureActivity
| where TimeGenerated > ago(1h)
| where OperationNameValue contains "resourcegroups/delete"
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue
```

> **If `AzureActivity` returns nothing — or errors with "table/entity not found":** ingestion has not finished. The empty result and the not-found error are the *same* condition (the first records have not landed on this new workspace yet), not a mistake in your query. Wait a few more minutes and re-run. This delay is the Step 3 lesson made real.

**What is KQL, and why does the whole industry keep meeting it?** **KQL** (Kusto Query Language) is the read-only query language for Log Analytics, and it powers Azure Monitor, Microsoft Sentinel, Defender, and more — learn it once and it pays off across every Microsoft observability tool. Read the first query as a pipeline, left to right, each `|` transforming the table: *take `AzureActivity`, keep the last day, count rows grouped by operation and status, sort descending.* That shape — **table, then a series of piped operators** (`where`, `summarize`, `project`, `order by`) — is the entire grammar. It reads like a sentence because it is designed to: filter, then aggregate, then shape.

**Why is querying so much more powerful than the filtered Activity Log blade you started with?** Because the blade lets you *filter*; KQL lets you *compute*. "How many operations of each type, grouped by success or failure, over the last day" is a question the blade cannot answer and one line of KQL can. Scale that up — join VM performance against deployment events, aggregate failures by caller, chart errors over time — and you see why serious operations teams live in this query window. The second query is the everyday version: a targeted hunt that found your Step 4 delete, stamped with who did it (`Caller`) and when. That is the exact capability your lead asked for — "I want to query it."

> **Checkpoint:** The first query returns a summary of your recent operations; the second surfaces your deliberate resource-group delete with your account as the `Caller`.

---

## Step 6 – Get Told Automatically: an Action Group and an Alert

Querying is pull — you have to go look. Your lead also wants push: to *be told*. That takes two pieces, an **action group** (who to notify and how) and an **alert rule** (the condition that triggers it).

First, the action group, via CLI:

```bash
az monitor action-group create \
  --resource-group rg-monitor-<yourinitials> \
  --name ag-catalog-oncall \
  --short-name catalcall \
  --action email primary <your-email-address>
```

Now create the alert rule in the portal (guided and clearer than the CLI for a first one):

1. Search for **Alerts** → **+ Create** → **Alert rule**.
2. **Scope:** select your **subscription**.
3. **Condition:** choose signal type **Activity Log**, and pick the signal **"Delete Resource Group"** (search the signal list for *delete resource group*).
4. **Actions:** select the existing action group **ag-catalog-oncall**.
5. **Details:** name it `alert-rg-deleted`, and set the **Resource group** to `rg-monitor-<yourinitials>` so it is cleaned up with everything else. Create the rule.

**Why separate the *who to notify* from the *what triggers it*?** Because the two change independently, and reuse is the payoff. An **action group** is a reusable notification target — an email, an SMS, a webhook to Slack, a call to an automation runbook — defined once. An **alert rule** is a condition watched over time. One action group (`ag-catalog-oncall`) can be wired to *many* alert rules — resource deleted, budget exceeded, VM CPU pegged — so when the on-call rotation changes, you update one action group instead of every rule. It is the same "define once, reference many" discipline you saw with RBAC groups in the identity lab, applied to notifications.

**Why alert on "Delete Resource Group" specifically?** Because it is the highest-consequence, lowest-frequency event in this whole course — the Day 1 lock existed precisely to prevent an accidental one, and deleting a group destroys everything inside it. It is a textbook thing to be *told* about immediately rather than discover later. More broadly, good alerting targets the rare-and-serious, not the routine: an alert that fires constantly gets ignored, so you reserve alerts for events that genuinely warrant a human interrupting their day. (You can leave this rule in place after class; on the free tier it costs nothing and is genuinely useful.)

> **Checkpoint:** The action group `ag-catalog-oncall` exists with your email, and the `alert-rg-deleted` rule references it. (Optionally: recreate and delete a throwaway resource group and, within a few minutes, expect an email — Activity Log alerts have the same ingestion latency as the logs.)

---

## Step 7 – Is It Me or Is It Azure? Service Health and Resource Health

When something looks wrong, the first question is whose fault it is. Two portal pages answer it.

1. Search for **Service Health**. This reports **Azure's** status *for your subscription and regions specifically* — active outages, planned maintenance, and health advisories affecting the services you actually use. It is not the generic public status page; it is filtered to you.

2. Search for **Resource Health** (or open any resource's **Resource health** blade). This reports whether a *specific resource* is healthy from Azure's side, and distinguishes "unavailable because of a platform issue" from "unavailable because of something you did."

**Why do these two pages matter as much as your own metrics?** Because they answer the "me or them?" fork that your lead named directly, and answering it wrong wastes hours. If your app is unreachable, you can burn an afternoon debugging your own code — or you can check Service Health, see that the region has a declared networking incident, and know to wait and communicate rather than fix. **Service Health** covers the platform broadly ("is Azure having a bad day in East US?"); **Resource Health** covers one resource ("is *this* VM's problem Azure's or mine?"). Together they draw the shared-responsibility line from Day 1 in real time: they tell you when the problem is on Microsoft's side of it, so you look in the right place first.

---

## Step 8 – The Standing Review: Azure Advisor

Search for **Advisor** and open it (you may have glimpsed it in the governance lab). Skim the recommendations across **Cost**, **Security**, **Reliability**, **Operational excellence**, and **Performance**.

**Why does Advisor belong in the monitoring lab too?** Because monitoring is not only *reactive* (alert me when it breaks) — it is also *proactive* (tell me what is likely to break, or waste money, before it does). Advisor is Azure continuously monitoring your environment against best practices and handing you a prioritized to-do list: an idle resource draining money, a workload with no redundancy, a security gap. Alerts catch the acute event; Advisor catches the slow-burning risk. A mature operations practice uses both — and Advisor's recommendations will frequently name the exact trade-offs you reasoned about all course (idle-resource cost, single points of failure, security hardening), which is a fitting note to end on.

> **Checkpoint:** You can state, in one sentence each, what Service Health, Resource Health, and Advisor each tell you — and why all three differ from an alert.

---

## Step 9 – Clean Up

Remove the monitoring resources:

```bash
az monitor diagnostic-settings subscription delete --name activity-to-la --yes

az group delete --name rg-monitor-<yourinitials> --yes --no-wait
```

> If you want to keep the `alert-rg-deleted` rule and its action group as a genuinely useful standing safeguard, they live in `rg-monitor-<yourinitials>` — either move them to a keeper resource group first, or accept they will be deleted with the group. Both cost nothing to keep.

**Why delete the diagnostic setting separately?** Because, like the policy assignment and the role assignment in earlier labs, the subscription diagnostic setting is attached at the *subscription* scope, not stored inside your resource group — deleting the group would leave it dangling, pointing at a workspace that no longer exists. This is the third time this pattern has appeared (policy assignments, subscription-scoped settings, Entra objects): governance and configuration objects that attach at a scope *above* the resource group need explicit removal. Recognizing that class of object on sight is a real operational skill.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Azure Monitor** | The umbrella over metrics, logs, and traces — the observability counterpart to ARM |
| **Activity Log** | The control-plane audit trail from Day 1, now used as a queryable data source |
| **Log Analytics workspace** | The consolidated, queryable store that most of Azure Monitor runs on |
| **Diagnostic setting** | Explicit routing: send these log categories from this source to that destination |
| **Ingestion latency** | Monitoring shows the recent past, not the live instant — which is why alerts matter |
| **KQL** | The pipe-based query language behind Monitor, Sentinel, and Defender — learn once, reuse everywhere |
| **Action group** | A reusable "who to notify, how" — defined once, referenced by many alert rules |
| **Alert rule** | A watched condition; target the rare-and-serious, not the routine |
| **Service Health / Resource Health** | Whether a problem is Azure's or yours — the "me or them?" answer |
| **Azure Advisor** | Proactive, standing best-practice review across cost, security, reliability, performance |

---

## Reflection

Answer for yourself or discuss with a partner:

1. You first met the Activity Log in Day 1 to prove one control plane. Here you used it as a monitoring data source. What did routing it into Log Analytics let you do that the Activity Log blade could not?
2. Ingestion latency meant your query did not show events instantly. Why does that latency make *alerting* important rather than just querying?
3. An action group and an alert rule are separate objects. Give one concrete reason that separation pays off.
4. Your app is down. Walk through the order in which you would check Resource Health, Service Health, and your own logs — and why that order.
5. Alerts and Advisor are both "monitoring." How do their jobs differ, and why do you need both?

---

## Optional Challenge

Two independent challenges, pick either or both:

1. **Write a sharper KQL query.** In your workspace Logs, write a query that returns only *failed* control-plane operations in the last day, grouped by `Caller` — `AzureActivity | where ActivityStatusValue == "Failure" | summarize count() by Caller`. Explain how, in a real breach or outage investigation, "failed operations by caller" is one of the first questions you would ask.
2. **Alert on something quieter.** Create a second alert rule for a lower-consequence but useful event — for example a **policy assignment** being created or a **role assignment** change (both appear as Activity Log signals). Then articulate the judgment call: which events deserve an interrupt-a-human alert, and which are better left to a dashboard you check on your own schedule? Why does over-alerting defeat the purpose?

(Clean up any extra rules and settings afterward.)

---

## Conclusion

You answered your lead's last and most important question — "how would we *know*?" You took the Activity Log you first met on Day 1 and turned it into observability: routed into a workspace, queried with KQL, and wired to an alert that would email you the instant something as serious as a resource-group deletion occurred. You learned where to look when the fault might be Azure's rather than yours, and where to find the standing advice that catches slow risks before they bite.

That closes the course. Across two days you moved from renting your first empty resource group to running a small but complete cloud practice: you can navigate and organize Azure, choose compute and design the network around it, give it real storage and data, control who touches it and where its secrets live, enforce standards the platform will not let anyone violate, and — now — watch all of it and be told when it needs you. Every one of those capabilities was enforced or surfaced at the same control plane you met on the first afternoon. That single idea — one place where the rules live and the signals gather — is the throughline of Azure, and you have now used it end to end.
