# Explore Azure's Structure – Regions, Hierarchy, and the Control Plane

## The Scenario

Your first assignment at **Cascade Outfitters** went well: you can navigate the portal, drive the CLI, and create resource groups. Now the catalog project is getting real, and your lead has three questions before the team commits to anything:

> "First — which region should the catalog run in? Don't guess; check what is actually available and what it costs. Second — set up the project's resource group properly, with tags, so finance and security stop asking me who owns what. Third — last year an intern deleted a production resource group. Make sure that cannot happen to ours."

This lab answers all three. Along the way you will see the two structures from the slides working together:

- The **physical layout** — geographies, regions, availability zones — which decides where things run, what they cost, and what rules they satisfy.
- The **logical hierarchy** — management groups, subscriptions, resource groups — which decides how things are billed, governed, and protected.

And you will finish with the most important experiment of the day: applying a **resource lock** and attacking it from two different tools. When both the portal and the CLI are refused by the same rule, you are watching Azure Resource Manager enforce policy at the control plane — the single most useful thing to understand about how Azure works.

As before, nothing here bills as long as you finish the cleanup step. Resource groups and locks are both free.

### Prerequisites

- Completion of the *Azure First Steps* lab, or comfort signing in to the portal and opening Cloud Shell
- An Azure account (the free account is enough)

---

## Step 1 – Check Service Availability by Region

Open **https://azure.microsoft.com/explore/global-infrastructure/products-by-region/** (or search the web for *Azure products by region*).

1. Pick a core service you have heard of — **Virtual Machines** or **Storage Accounts** — and look at how many regions offer it.
2. Now pick a newer or more specialized service (browse the AI or analytics categories) and compare its region list.

**Why check this before choosing a region?** Because one of the four factors in region choice is **service availability**, and beginners routinely assume every service exists everywhere. It does not. Core services are nearly universal; newer and specialized services often launch in a handful of regions and expand slowly. The classic mistake this step prevents: a team builds everything in the region closest to their office, then discovers three months in that a service they need is not offered there — and moving is painful. Thirty seconds on this page avoids that.

**What are the other three factors?** Proximity to users (latency), data-residency and compliance rules (the *geography* boundary), and price. You will see the price factor with your own eyes in Step 3.

---

## Step 2 – List the Regions You Can Actually Deploy To

Open **Cloud Shell** (the `>_` icon, Bash) and run:

```bash
az account list-locations --query "[].{Region:name, Display:displayName}" --output table
```

Scroll the list. There are more than 70 regions.

**Why look at this list in the CLI when the map on the slides was prettier?** Because these names — `eastus`, `westeurope`, `southeastasia` — are the **exact strings you type when you deploy**. The map is for understanding; this list is for doing. Notice the pattern in the names: a geography-flavored word plus a direction. When a command later asks for `--location`, the answer comes from this list.

**Why did we use `--query` again?** The raw output of `list-locations` includes metadata you do not need yet (coordinates, region categories, pairing information). The query keeps the two columns that matter today. Some of what it filtered out is worth seeing, though — that is the next command.

Now look inside one region. Ask for everything Azure knows about `eastus`:

```bash
az account list-locations --query "[?name=='eastus']"
```

Read the JSON and find two things:

1. **`availabilityZoneMappings`** — a list of zones (`eastus-az1`, `eastus-az2`, `eastus-az3`). These are the **availability zones** from the slides: physically separate datacenters inside the region, each with independent power, cooling, and networking.
2. **`pairedRegion`** — for East US, the value is `westus`. This is the region's built-in **region pair** for geo-redundancy.

**Why do zones and pairs both exist — aren't they the same idea?** They protect against different sizes of disaster, and the difference is distance. **Zones** are separate buildings a few miles apart inside one region: deploy across them and a datacenter fire or power failure cannot take you down, while latency stays low enough for synchronous work. **Pairs** are whole regions hundreds of miles apart: replicate to the pair and even a hurricane that takes out the entire region cannot destroy your data. Zones are for *high availability* (staying up through a local failure); pairs are for *disaster recovery* (surviving a regional one). You will not deploy zonal resources today, but every resilience conversation you have in Azure will use these two words — and now you have seen where they live in the metadata.

**One related term to keep straight:** an **availability set** is an older, smaller-scope mechanism that spreads VMs across racks *within one datacenter* — protection against a rack or maintenance event, not a building failure. Zones superseded sets for most new designs; you will mostly meet sets in existing systems.

---

## Step 3 – Watch the Price Change with the Region

Open the **Azure Pricing Calculator** at **https://azure.microsoft.com/pricing/calculator/**.

1. Add a **Virtual Machines** item.
2. Pick a small size (a D2 v5 or similar is fine).
3. Switch the **Region** dropdown between **East US** and a region in Europe or Asia. Watch the monthly estimate change.

**Why does the identical VM cost different amounts in different places?** Because a region is real buildings drawing real electricity on real land, and energy, real estate, and demand vary by country. Azure's prices reflect those local costs. You do not need to memorize any numbers — the habit is the point: when a region choice is open, check **availability** first, then **latency**, **compliance**, and **price**. Teams with flexible workloads sometimes save meaningfully just by choosing a cheaper nearby region.

> **Checkpoint:** You can name one service that is not available in every region, you found East US's three availability zones and its paired region in the metadata, and you have seen the same VM priced differently in two regions. You can now answer your lead's first question.

---

## Step 4 – See Where You Sit in the Hierarchy

In the portal, search for **Management groups** and open it.

Depending on your account's permissions you may see a *Tenant Root Group*, or a prompt about registering — either is fine. Do not create anything; just observe. Then search for **Subscriptions** and note that your subscription is the level below.

**Why look at a level you will not touch today?** Because the logical hierarchy from the slides — management groups contain subscriptions, subscriptions contain resource groups, resource groups contain resources — only makes sense when you can place *yourself* in it. In this course you work inside one subscription, so resource groups are your everyday level. But in a real enterprise, the rules that govern you (policy, access, spending limits) are usually set one or two levels *above* you and inherited downward. Knowing the levels exist explains where those rules come from — and why you sometimes cannot override them.

**Why do organizations split subscriptions at all?** Two boundaries live at the subscription level: **billing** (each subscription gets its own bill) and **quotas** (each subscription has resource limits). Splitting by environment or business unit keeps one team's spending and one team's quota consumption from hiding inside another's.

---

## Step 5 – Build the Project's Resource Group, Properly Tagged

Now for your lead's second question. In Cloud Shell, run (use your initials):

```bash
az group create \
  --name rg-catalog-<yourinitials> \
  --location eastus \
  --tags project=catalog environment=dev owner=<yourinitials> cost-center=retail-web
```

**Why four tags this time instead of one?** Because these four answer the four questions every organization eventually asks about every resource:

| Tag | The question it answers | Who asks it |
|---|---|---|
| `project` | What is this for? | Everyone |
| `environment` | Can I break it? (`dev` yes, `prod` no) | Engineers |
| `owner` | Who do I call about it? | Security, operations |
| `cost-center` | Whose budget pays for it? | Finance |

Cost reports slice by tags, access reviews filter by tags, and cleanup scripts target tags. An untagged resource can answer none of these questions, which is why mature organizations *require* tags through policy — a topic you will meet on Day 2. Applying them at creation is one flag on a command; applying them across a thousand existing resources is a project.

---

## Step 6 – Read It Back, Two Ways

First the summary:

```bash
az group show --name rg-catalog-<yourinitials> --query "{name:name, location:location, tags:tags}" --output json
```

Then the full ARM view:

```bash
az group export --name rg-catalog-<yourinitials>
```

The second command prints a JSON **deployment template** of everything *inside* the group — and look closely: `"resources": []`. The template is empty, because the group is empty.

**Why read back what you just created?** Verification is a reflex worth building: create, then confirm, every time. In the `az group show` output, notice the group has exactly one `location` for its metadata even though future contents could live elsewhere — the gotcha from the slides, visible in the JSON.

**Why look at an empty template?** Because even empty, it is a first glimpse of **infrastructure as code**. Every Azure resource is, underneath everything, a JSON document that Azure Resource Manager knows how to realize, and `export` writes that document *from* whatever exists. Your group's recipe has no ingredients yet — run this same command against the resource group you build in this afternoon's networking lab and the template will describe every VM, subnet, and security rule in it. An ARM template or Bicep file is that same document written *before* the resources exist, so a whole environment can be deployed from a file. You will not author templates today — just remember that the portal's forms and the CLI's flags are both ways of filling in this JSON.

> **Checkpoint:** `az group show` returns your group with all four tags.

---

## Step 7 – Lock the Group Against Deletion

Your lead's third question: make the intern incident impossible. Apply a **resource lock**:

```bash
az lock create \
  --name protect-catalog \
  --lock-type CanNotDelete \
  --resource-group rg-catalog-<yourinitials>
```

Now open the resource group in the portal and click **Locks** in its left menu. The lock you just created from the CLI is sitting there.

**Why do locks exist when permissions already control who can do what?** Because the intern *had* permission — that was the problem. Deleting a resource group deletes everything inside it, and the people with the power to do that deliberately also have the power to do it accidentally: a stray click, a script pointed at the wrong name, a tired Friday afternoon. A lock does not manage *who*; it protects the resource from *everyone*, including its owners, until someone deliberately removes the lock. It converts a one-step disaster into a two-step deliberate act.

**Why `CanNotDelete` and not `ReadOnly`?** There are two lock types, and they answer different threats:

| Lock type | Allows | Blocks | Use it when |
|---|---|---|---|
| `CanNotDelete` | Reading and modifying | Deletion | The resource should evolve but never vanish |
| `ReadOnly` | Reading | Modifying *and* deleting | Nothing should change at all — beware, this can break services that write to their own settings |

The catalog project is under active development, so the team must be able to change things. Only deletion needs to be off the table. That is `CanNotDelete`.

---

## Step 8 – Attack the Lock from the Portal

Time for the experiment. In the portal, open `rg-catalog-<yourinitials>`, click **Delete resource group**, type the name to confirm, and attempt the delete.

**Expected result: it fails.** You get an error stating the resource group (or its scope) is locked and the lock must be removed before deleting.

Read the error message before moving on — notice it identifies the locked scope (your resource group) and tells you a lock must be removed first.

---

## Step 9 – Attack the Lock from the CLI

Now try the other door. In Cloud Shell:

```bash
az group delete --name rg-catalog-<yourinitials> --yes
```

**Expected result: it also fails**, with the same kind of lock error.

**Why is this the heart of the lab?** You attacked the same resource from two completely different tools, and both were stopped by the same lock with the same explanation. Neither tool has special power over the other, because neither tool is where the rule lives. The portal and the CLI are both just **clients of Azure Resource Manager**, and the lock is enforced at the ARM layer — after the request arrives, before anything is touched, identically for every caller.

This is why you cannot "get around" a lock, a permission, or a policy by switching tools. It is also why governance in Azure scales: a rule stated once at the control plane binds the portal, the CLI, PowerShell, SDKs, and every automation pipeline, without any of them cooperating. Change the rule once; every door updates.

> **Checkpoint:** Both delete attempts were refused with a lock error.

---

## Step 10 – Remove the Lock, Then Clean Up

Removing the lock is the deliberate second step that makes deletion possible again:

```bash
az lock delete --name protect-catalog --resource-group rg-catalog-<yourinitials>
```

Now the delete goes through:

```bash
az group delete --name rg-catalog-<yourinitials> --yes --no-wait
```

Confirm it is gone (or going):

```bash
az group list --output table
```

**Why does the two-step dance matter?** Notice what just happened: to destroy the group you had to perform two distinct, intentional actions — unlock, then delete. No single mistake could have done it. That *is* the protection. In production, teams often go further and restrict lock-removal permission to a smaller group than resource-modification permission, so the second step requires a second person.

**Why clean up a free resource group again?** Same habit as the first lab: nothing here bills, but the reflex must be automatic *before* you start creating resources that do — which happens in the very next lab. Cleanup is on the customer side of the shared responsibility line; Azure will not tidy up after you.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Service availability by region** | Not every service is everywhere — check before committing to a region |
| **Region names** | `eastus`-style strings from `az account list-locations` are what you deploy with |
| **Availability zones** | Physically separate datacenters inside a region — deploy across them to survive a building failure |
| **Region pairs** | A partner region hundreds of miles away — replicate to it to survive a regional disaster |
| **Zones vs sets vs pairs** | Rack-level, building-level, and region-level protection — match the mechanism to the failure you fear |
| **Regional pricing** | The same VM costs different amounts in different regions |
| **The logical hierarchy** | Management groups → subscriptions → resource groups; rules set above you flow down |
| **Tags as governance** | `project`, `environment`, `owner`, `cost-center` answer the four questions every org asks |
| **`az group export`** | Every resource is JSON underneath — the seed of infrastructure as code |
| **Resource locks** | `CanNotDelete` blocks deletion for everyone until deliberately removed |
| **ARM enforcement** | The portal and the CLI were refused by the same rule, because rules live at the control plane, not in tools |

---

## Reflection

Answer for yourself or discuss with a partner:

1. You could not delete the locked group from either the portal or the CLI. In one sentence, why not?
2. A teammate has permission to modify but not delete a resource. Would switching from the portal to the CLI let them delete it? Why or why not?
3. Your company must keep customer data in Canada, wants the lowest latency for users in Vancouver, and needs a service that is only available in `canadacentral`. Walk through how the four region factors settle the choice.
4. Give one real situation where you would choose a `ReadOnly` lock over `CanNotDelete` — and one risk of doing so.

---

## Optional Challenge

Create a second resource group and apply a **`ReadOnly`** lock instead of `CanNotDelete`. Before touching anything, predict: will adding a tag to the group succeed or fail? Test your prediction with `az group update --name <name> --set tags.test=true`. Then explain the result in terms of what `ReadOnly` blocks. Remove the lock and delete the group when done.

---

## Conclusion

You answered all three of your lead's questions. You compared regions on availability and price instead of guessing, you built a properly tagged resource group that finance and security can trace, and you made accidental deletion impossible — then proved it by attacking the lock from two tools and losing both times, to the same referee.

That referee, Azure Resource Manager, is the idea to carry forward: every rule you meet in Azure — locks today, RBAC and Azure Policy on Day 2 — is enforced at the control plane, uniformly, for every tool. Next you will stop making empty containers and start filling one: the catalog application gets deployed, twice, in two very different service models.
