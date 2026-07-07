# Enforce the Rules Before They're Broken – Governance and Cost

## The Scenario

Every lab so far, you have tagged resources by hand — `project`, `environment`, `owner` — because it is good practice. Back in Day 1 you read a promise: *"mature organizations require tags through policy — a topic you will meet on Day 2."* This is that lab. Your lead has watched the tagging habit slip on every team they have ever been on, and they are done relying on discipline:

> "Good intentions don't survive a busy Friday. Half our resources will end up untagged if the only thing stopping that is people remembering. I want the *platform* to refuse an untagged resource — no owner tag, no deployment. And while you're at it, show me we won't get a surprise bill. I've been burned by finding out about spend a month too late."

That is **governance**: encoding organizational standards so the platform enforces them automatically, instead of hoping humans will. And its partner is **cost management**: seeing and bounding spend before the invoice arrives. In this lab you will assign an **Azure Policy** that *denies* the creation of any resource lacking an `owner` tag, watch a non-compliant deployment get refused, then watch a compliant one succeed. Then you will tour the cost tools — Cost analysis, budgets, the pricing calculator, and Azure Advisor — that keep spend visible and bounded.

The refusal you are about to trigger is the same control-plane enforcement you have met all course — the lock in Day 1, the analyst's Reader role in the last lab. Policy is that same idea aimed at a new target: not *who* may act, but *what* they are allowed to create.

> **Cost note:** Policy assignments, budgets, and the tools you tour are all free. The one storage account you try to create costs nothing (the non-compliant attempt is refused; the compliant one is deleted in cleanup).

### Prerequisites

- Completion of Day 1, or comfort with Cloud Shell, resource groups, and tags
- An Azure account. Assigning policy requires **Owner** or **Resource Policy Contributor**; creating a budget requires **Owner** or **Cost Management Contributor**. A personal/free subscription gives you all of this.

---

## Step 1 – Create a Resource Group to Govern

```bash
az group create \
  --name rg-governance-<yourinitials> \
  --location eastus \
  --tags project=catalog environment=dev owner=<yourinitials>
```

**Why govern at the resource-group scope in this lab?** Because scope works for policy exactly as it did for RBAC in the last lab: a rule set at the subscription flows down to every resource group; a rule set at one resource group affects only what is inside it. In production you would usually assign a tagging policy high up — at a management group or the subscription — so it covers everything. Here you scope it to one resource group so your experiment cannot interfere with the rest of your subscription while you learn how the enforcement behaves.

---

## Step 2 – Assign a Policy That Requires an `owner` Tag

Azure ships hundreds of **built-in** policy definitions. One of them, *"Require a tag on resources,"* denies creation of any resource missing a named tag. Assign it to your resource group, configured to require `owner`:

```bash
RG_ID=$(az group show --name rg-governance-<yourinitials> --query id -o tsv)

az policy assignment create \
  --name require-owner-tag \
  --display-name "Require an owner tag on resources" \
  --policy "871b6d14-10aa-478d-b590-94f262ecfa99" \
  --scope "$RG_ID" \
  --params '{ "tagName": { "value": "owner" } }'
```

**What is Azure Policy, and how is it different from RBAC?** RBAC controls *who* can act; Policy controls *what the resulting resource is allowed to look like*, regardless of who is acting. The analyst's Reader role in the last lab said "this person may not create resources." This policy says "*nobody* — not even you, the Owner — may create a resource here without an `owner` tag." They are complementary halves of governance: identity rules and configuration rules, both enforced at the control plane. A policy definition describes a *condition* (does the resource have an `owner` tag?) and an *effect* (if not, **Deny**). Other effects exist — **Audit** (allow but flag it), **Append** or **Modify** (add the missing tag automatically), **DeployIfNotExists** (create a companion resource) — but Deny is the one that makes "no tag, no deployment" literal.

**Why use a built-in definition instead of writing one?** Because Microsoft maintains a large library of common rules — require tags, restrict regions, allow only certain VM sizes, enforce encryption — and reaching for a built-in is faster and less error-prone than authoring policy JSON. The long GUID (`871b6d14-...`) is the built-in definition's ID; `--params` supplies the one value it needs, the tag name. When you eventually need a rule Microsoft does not ship, you can write a custom definition in the same JSON shape — but reach for built-ins first.

**Why does policy relate to the tagging you have done all course?** Because tags are only as useful as they are *complete*. A cost report sliced by `owner` is worthless if half the resources have no owner tag; an "find everything project=catalog" search misses whatever nobody remembered to label. Policy is what turns tagging from a hopeful convention into a guarantee — which is exactly why the Day 1 lab told you this moment was coming.

> **Note:** A new policy assignment usually begins enforcing Deny within a few minutes, but Azure allows it up to ~30 minutes to fully propagate. If your Step 3 attempt is *allowed* through, that is propagation lag, not a mistake on your part — read ahead, then retry the attempt periodically until it is refused. (This is a good moment to skim Steps 6–7 while you wait.)

---

## Step 3 – Try to Break the Rule (and Watch It Refuse)

Attempt to create a storage account in the governed resource group **without** an `owner` tag:

```bash
az storage account create \
  --name stnotag<yourinitials> \
  --resource-group rg-governance-<yourinitials> \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

**Expected result: it fails.** The error is a `RequestDisallowedByPolicy` message naming your `require-owner-tag` assignment and explaining the resource was denied for missing the required tag.

**Why is this the heart of the lab?** Because the platform just refused *you* — a subscription Owner, with every permission — from creating a resource, not because you lacked access, but because the resource itself violated a standard. That is governance working the way your lead wants: the rule does not depend on anyone remembering it, and it does not depend on who is deploying. Read the error closely; it names the policy, which is exactly what a teammate hitting this in six months needs to understand why their deployment bounced. And notice *where* the refusal happened — at Azure Resource Manager, before the storage account was ever created. This is the same control plane that enforced the Day 1 lock and the last lab's Reader role, now enforcing a configuration standard. One enforcement point, three kinds of rule.

> **Checkpoint:** The storage account creation is refused with a policy error that names `require-owner-tag`.

---

## Step 4 – Comply with the Rule (and Watch It Succeed)

Now create the same storage account *with* the required tag:

```bash
az storage account create \
  --name stnotag<yourinitials> \
  --resource-group rg-governance-<yourinitials> \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --tags owner=<yourinitials>
```

**Expected result: it succeeds.** The only thing that changed was adding `--tags owner=<yourinitials>`.

**Why run the near-identical command twice?** Because the contrast *is* the lesson, exactly as creating a resource group two ways was on Day 1. Same person, same permissions, same resource, same region — the only variable is whether the resource meets the standard, and that single variable decided allow versus deny. That is what "the platform enforces the rule" means in practice: compliance, not identity or intent, is the gate. A developer who tags their resources never notices the policy exists; one who forgets is stopped at the door with an explanation. The rule scales to a thousand engineers because none of them have to know it is there.

> **Checkpoint:** With the `owner` tag present, the storage account is created successfully.

---

## Step 5 – Review Compliance State

See how the policy views your resource group now:

```bash
az policy state summarize --resource-group rg-governance-<yourinitials> \
  --query "value[0].results" --output json
```

(If it reports no results yet, compliance scanning runs periodically and may take a few minutes to populate — the *Deny* enforcement in Steps 3–4 was immediate regardless.)

**Why does a *deny* policy still track compliance?** Because Deny only governs *new* creations going forward — it cannot un-create the resources that already existed before the assignment, and many useful policies use the softer **Audit** effect precisely so they can report on an existing estate without blocking anyone. The compliance dashboard (here via CLI, and far richer in the portal under **Policy → Compliance**) is where governance becomes *visible*: a single view of what fraction of your resources meet each standard. For an organization with a dozen policies across thousands of resources, that dashboard is the difference between "we have standards" and "we can prove we meet them" — which is the entire point of the **Azure Blueprints**, **Microsoft Purview**, and compliance-offering tooling the course mentions: governance you can *demonstrate*, not just intend.

---

## Step 6 – See and Bound the Spend

Governance covered "what gets built." Now your lead's second worry: "no surprise bills." Tour the cost tools — most are read-only, so this is fast.

1. **Cost analysis.** Search for **Cost Management + Billing** → **Cost analysis** (or **Subscriptions → your subscription → Cost analysis**). This is the same page you found empty on Day 1; it slices actual spend by resource, by resource group, and — crucially — **by tag**. Group the view by the `owner` tag.

2. **Confirm the tagging payoff.** The policy you just assigned is *why* that tag-grouped cost view can ever be complete. An untagged resource shows up as "no owner" — an unaccountable line on the bill. Tag enforcement and cost visibility are the same story told twice: you cannot manage spend you cannot attribute.

3. **A budget with an alert.** If you did the Day 1 first-steps lab you set one; if not, under **Cost Management → Budgets → + Add**, create a small budget (e.g., $5) with an alert at 80% to your email. A budget does not *stop* spend — it emails you when actual or forecast cost crosses a line, turning "found out a month too late" into "found out the same day."

4. **Estimate before you build.** Open the **Azure Pricing Calculator** (`azure.microsoft.com/pricing/calculator`) and price a small VM; open the **TCO (Total Cost of Ownership) Calculator** and note it compares cloud cost against running the same workload on-premises. One tool answers "what will this cost to run in Azure?"; the other answers "is Azure cheaper than our datacenter?" — the CapEx-vs-OpEx question from Day 1, with numbers.

**Why are these four tools one discipline, not four features?** Because they cover the full timeline of a dollar: **estimate** before you commit (calculators), **bound** what you are willing to spend (budgets), and **watch** what you actually spent, attributed to the right team (Cost analysis, made trustworthy by tag policy). Miss any one and cost control has a hole — you either commit blind, spend unbounded, or cannot tell whose spend is whose. Your lead's fear lives in the gap between spending and finding out; these tools close it.

---

## Step 7 – Take One Free Round of Advice

Search for **Advisor** and open it. Azure Advisor scans your subscription and recommends improvements across five categories: **Cost**, **Security**, **Reliability**, **Operational excellence**, and **Performance**.

**Why end a governance lab with Advisor?** Because governance is not only rules you *impose* — it is also gaps you did not know you had. Advisor is a standing, automated review of your whole environment against Microsoft's best practices: idle resources you are paying for, security settings left open, single points of failure, workloads that could run cheaper. It is free, it updates continuously, and it is the closest thing to a second set of eyes on everything you have built across both days. Skim your recommendations; several will echo decisions from this course (cost of idle resources, security hardening), which is a satisfying sign the principles generalize.

> **Checkpoint:** You can name where to check spend by tag, where to set a spend alert, which calculator estimates Azure cost versus on-premises, and what Advisor is for.

---

## Step 8 – Clean Up

Remove the policy assignment and the resource group:

```bash
az policy assignment delete --name require-owner-tag --scope "$RG_ID"

az group delete --name rg-governance-<yourinitials> --yes --no-wait
```

**Why delete the policy assignment explicitly when deleting the resource group?** Because a policy assignment is scoped *to* the resource group but is not a resource *inside* it — it is a governance object attached at that scope, much like the Entra ID objects in the last lab lived outside the resource hierarchy. Deleting the group removes the storage account within it, but tidy governance means removing the assignment you created too, so no orphaned rule lingers pointing at a scope that no longer exists. Leave the budget in place if you like — a standing spend alert is a good habit, and it costs nothing.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Azure Policy** | Rules about *what* a resource may be, enforced at the control plane regardless of who acts |
| **Policy vs RBAC** | RBAC governs *who* can act; Policy governs *what the result may look like* — complementary halves of governance |
| **Effects** | Deny, Audit, Append/Modify, DeployIfNotExists — Deny makes "no tag, no deployment" literal |
| **Built-in definitions** | Microsoft's library of ready-made rules; reach for these before writing custom policy |
| **Scope** | Policy inherits down the hierarchy exactly like RBAC — assign at the altitude the rule should cover |
| **Compliance state** | The dashboard that turns "we have standards" into "we can prove we meet them" |
| **Cost analysis by tag** | Attributed spend — only trustworthy when tag policy makes tagging complete |
| **Budgets** | Alert on spend before the invoice, not after |
| **Pricing / TCO calculators** | Estimate Azure cost, and compare Azure against on-premises (OpEx vs CapEx) |
| **Azure Advisor** | A free, continuous best-practice review across cost, security, reliability, and performance |

---

## Reflection

Answer for yourself or discuss with a partner:

1. The same storage account command was denied, then allowed, with only a tag added. Who was stopped the first time, and why is "even the Owner" the important part?
2. Policy and RBAC both refused an action in this course. State the difference between what each one governs, in one sentence.
3. Why is a tag-enforcement policy a *prerequisite* for trustworthy cost reporting, rather than an unrelated nicety?
4. A budget alert fires at 80% but nothing stops running. Why is "alert, don't block" the right behavior for a budget, and what would you use if you genuinely needed spend to halt?
5. You chose the **Deny** effect. When would **Audit** be the wiser choice for the very same tagging rule?

---

## Optional Challenge

Two independent challenges, pick either or both:

1. **Restrict regions with policy.** Assign the built-in *"Allowed locations"* policy to your resource group, permitting only `eastus`. Then try to create a resource in `westus` and watch it be denied. Explain how region-restriction policy enforces the data-residency requirement you reasoned about on Day 1 — automatically, instead of by trusting everyone to pick the right region.
2. **Auto-fix instead of refuse.** Research the difference between the **Deny** and **Modify** effects for tags. Explain a scenario where you would rather the platform *add* a missing tag (with a default value) than reject the deployment — and one risk of doing so silently.

(Clean up any extra assignments and resources afterward.)

---

## Conclusion

You made good on the Day 1 promise: tagging is no longer a hopeful convention on your team, it is a rule the platform enforces — no owner tag, no deployment, not even for the Owner. And you closed the loop your lead cared about most, connecting that enforcement to trustworthy cost reporting: because every resource is tagged, every dollar can be attributed, and budgets plus calculators bound and forecast the spend so nobody finds out a month too late.

Notice the shape of the whole day so far: a lock protecting a resource, a role limiting a person, a policy constraining a deployment — three different rules, one control plane refusing what violates them, every time, through every tool. In the final lab you turn from *controlling* the environment to *observing* it: Azure Monitor, Log Analytics and KQL, alerts, and Service Health — the instrumentation that tells you what everything you have built across both days is actually doing, and warns you the moment it misbehaves.
