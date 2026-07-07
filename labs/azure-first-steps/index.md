# Azure First Steps – The Portal, Cloud Shell, and Your First Resource

## The Scenario

You have just joined the cloud team at **Cascade Outfitters**, a retailer that is moving its product catalog application to Azure. The company has a brand-new Azure subscription and nothing in it. Before anyone deploys a single virtual machine, your lead gives you a first assignment:

> "Get into the subscription, learn your way around, and prove to yourself that the portal and the command line are the same Azure. Then create the resource group the catalog project will live in — and show me you can find where our money goes."

That assignment is this lab. Along the way you will confirm three ideas from the slides with your own hands:

- **The cloud is renting computing and paying for use** — you will find the page where the meter is read.
- **A resource group is the container everything lives in** — you will create one three different ways.
- **Every tool talks to the same control plane (Azure Resource Manager)** — you will watch the portal, the CLI, and PowerShell produce identical results, and see all of it recorded in the same activity log and found by the same tag search.

Nothing in this lab costs money as long as you complete the cleanup step. A resource group is free — it only holds things; it does not bill.

### Prerequisites

- A computer with a modern web browser
- An Azure account — the free account created earlier, or the account your instructor provided
- No prior Azure experience is required

---

## Step 1 – Sign In to the Azure Portal

Go to **https://portal.azure.com** and sign in with your Azure account.

**Why start with the portal instead of the command line?** The portal is the web front door to Azure, and it is the best place to *learn* because you can see everything: every service, every setting, every price. But it is only one of several doors into the same platform. By the end of this lab you will have used two of them and confirmed they lead to the same place. Professionals tend to graduate from the portal to the CLI as their work becomes repetitive — you will feel why later in this lab.

---

## Step 2 – Find the Four Landmarks

Take a few minutes and locate each of these. Do not change anything yet — just find them.

| Landmark | Where it is | What it is for |
|---|---|---|
| **Global search bar** | Top center | The fastest way to reach any service or resource. Type `resource groups` and notice it finds both *services* and *your own resources*. |
| **Portal menu** | Three-line icon, top left | The list of all services, and the **Create a resource** button. |
| **Account and directory** | Your avatar, top right | Shows which account you are signed in as and which **directory** (also called a *tenant*) you are working in. Click it and read the directory name. (Your current **subscription** lives one step away — you will find it in the next step.) |
| **Cloud Shell icon** | Top toolbar, looks like `>_` | A terminal in your browser. You will open it in Step 4. |

**Why does this matter?** Almost every problem a beginner hits in Azure reduces to one of two things: *"I was in the wrong subscription"* or *"I could not find the thing."* Knowing these four landmarks prevents most of that. The subscription deserves special attention: it is both your **billing boundary** and a **limit on scale** (each subscription has resource quotas), so you should always know which one you are in before you create anything.

---

## Step 3 – Find Where the Money Goes

1. In the search bar, type **Subscriptions** and open it.
2. Click your subscription.
3. In the left menu, open **Cost analysis** (if Cost analysis is not available on your account type, open **Overview** instead).

Your cost right now should be at or near zero — you have not created anything that bills.

**Why look at a page full of zeros?** This is the consumption-pricing idea from the slides made concrete. In the cloud, the meter runs whenever a billable resource is allocated — whether or not it is doing useful work. This page is where the meter is read. When you build real resources later today, this is where you would catch a forgotten VM billing all weekend. Finding this page *before* you spend anything means you will never have to hunt for it in a panic.

**Why is the subscription where cost lives?** Because the subscription is Azure's billing boundary. Every resource you create belongs to exactly one subscription, and that subscription gets the bill. Organizations split subscriptions by team or environment precisely so this page answers "what is *that* team spending?" without any extra work.

**This page is also the CapEx-to-OpEx shift from the slides, made visible.** Cascade Outfitters used to buy servers — a capital expense committed years in advance, whether the hardware ran hot or sat idle. In Azure, this page *is* the spend: an operating expense that starts at zero, moves only when you allocate something, and stops when you clean up. That is why the CFO cares about a page you can check daily instead of a purchase order signed once every three years.

**Put a guardrail on it while you are here.** In the left menu, open **Budgets**, then **+ Add**. Give it a name like `catalog-training-budget`, set the amount to something small like **$5**, and set an alert condition at **80%** of that amount, with your own email as the recipient. Save it.

**Why add a budget on a subscription that is nowhere near spending $5?** Because the right time to set a guardrail is *before* you need it, not after a surprise bill. A budget does not stop spending — it is not a lock — but it emails someone the moment actual or forecasted cost crosses a threshold. In a real organization, budgets are how a team catches a runaway resource on day two of a mistake instead of day thirty, when the invoice arrives. You will not hit this $5 threshold today, but you now know exactly where you would set one on a project that matters.

> **Checkpoint:** You can say out loud which directory and subscription you are in, where you would go to check spending, and you have a budget alert configured (even though it will not fire today).

---

## Step 4 – Open Cloud Shell

Click the Cloud Shell icon (`>_`) in the top toolbar.

1. If prompted for a shell type, choose **Bash**.
2. If prompted to create storage the first time, accept the defaults.

> **Note:** The first launch may create a small storage account so your shell files persist between sessions. This is expected and costs only pennies per month. You can remove it after class if you wish.

**Why Cloud Shell instead of installing the CLI on your laptop?** Cloud Shell is a terminal that runs in your browser with the Azure CLI and Azure PowerShell already installed and already signed in as you. There is nothing to install, no versions to manage, and it works identically on every laptop in the room. It is also a live demonstration of a point from the slides: the command line and the portal are just **different doors into the same Azure**. You are about to prove that.

While you are here, notice two more icons in the Cloud Shell toolbar: a small **folder icon** (Manage files — upload and download files between your laptop and the storage account behind Cloud Shell) and a **pencil/brackets icon** (a built-in code editor for editing files without leaving the browser). You will not need either in this lab, since every file you create today comes from a short command pasted straight into the terminal, but later labs build multi-line configuration files, and knowing the editor is one click away saves you from fighting `nano` or `vi` syntax under time pressure.

---

## Step 5 – Confirm Who and Where You Are

In Cloud Shell, run:

```bash
az account show --output table
```

Compare the subscription name in the output to what you saw in the portal in Step 2. They should match.

**Why check this?** `az account show` prints the subscription the CLI is currently pointed at. It matches the portal because both tools are asking the same control plane the same question. This is your first piece of evidence that the front door does not change the answer. It is also a professional habit: running a destructive command against the wrong subscription is one of the classic cloud accidents, and this command is the thirty-second insurance against it.

**Why `--output table`?** The Azure CLI returns JSON by default, which is ideal for scripts but noisy for humans. The `--output table` flag (or `-o table`) formats results for reading. You will use both formats today — JSON when you want every detail, table when you want a quick look.

---

## Step 6 – List What Already Exists

Run:

```bash
az group list --output table
```

On a fresh subscription the list may be empty. That is fine — you are about to change it.

**Why list before creating?** Two reasons. First, it establishes a baseline: when your new resource group appears in this list a few minutes from now, you will know it came from you. Second, `az group list` is the command you will run constantly in real work to answer "what is in this subscription?" — worth meeting it early.

> **Checkpoint:** `az account show` returns your subscription, and `az group list` runs without error (an empty result is fine).

---

## Step 7 – Create a Resource Group in the Portal

This is the container the Cascade Outfitters catalog project will live in. First, build it the visual way.

1. In the search bar, type **Resource groups** and open it, then click **+ Create**.
2. On the **Basics** tab, enter or select the following:

   | Setting | Value |
   |---|---|
   | Subscription | Leave your subscription selected |
   | Resource group name | `rg-catalog-portal-<yourinitials>` (for example `rg-catalog-portal-js`) |
   | Region | A region near you (for example **East US**) |

3. Click **Next: Tags** and add one tag:

   | Name | Value |
   |---|---|
   | `course` | `azure-core-solutions` |

4. Click **Review + create**, then **Create**.
5. When the notification appears, click **Go to resource group** and look around. It is empty — a labeled box waiting for contents.

**Why does the name matter?** The name should tell a human what this is and who owns it, months from now, with no context. `rg-catalog-portal-js` says: it is a resource group, it belongs to the catalog project, it was made in the portal, and js owns it. Good naming is the cheapest form of governance there is — it costs nothing and saves hours.

**Why does a resource group need a region if it is just a container?** This is the gotcha from the slides: the region you picked only sets where the group's **metadata** (its record, its tags, its list of contents) is stored. The resources you later put *inside* it can live in completely different regions. You chose where the filing cabinet lives, not where every future resource must run.

**Why tag it now instead of later?** Tags are how organizations slice cost reports, find resources among thousands, and apply policy. An untagged resource is a resource nobody can account for. Tagging at creation time is a habit; tagging retroactively is a cleanup project.

---

## Step 8 – Create a Resource Group with the CLI

Now do the same thing the other way. In Cloud Shell, run (change the initials):

```bash
az group create \
  --name rg-catalog-cli-<yourinitials> \
  --location eastus \
  --tags course=azure-core-solutions created-by=cli
```

The command returns a block of JSON describing what it created. Read it — you will recognize the name, location, and tags you just supplied, plus a `provisioningState` of `Succeeded`.

**Why do the same task twice?** Because the comparison is the lesson. You created a resource group by clicking, and now by typing, and the result is the same kind of object governed by the same rules — because both requests went through **Azure Resource Manager**. Clicking is great for learning and one-off tasks. The command line is how you would create a hundred resource groups consistently — no missed fields, no typos, and the command itself can be saved, reviewed, and version-controlled. When you find yourself doing the same portal task a third time, that is the signal to script it.

**Why is the JSON output worth reading?** Because in Azure, *everything is an object with properties*. The portal shows you those properties as forms and blades; the CLI shows them raw. Getting comfortable reading resource JSON now pays off tomorrow when you meet ARM templates — which are exactly this JSON, written *before* the resource exists.

---

## Step 9 – Do It a Third Way: Azure PowerShell

Cloud Shell speaks two languages. So far you have used Bash and the Azure CLI (`az`). Switch to the other one.

In the Cloud Shell toolbar, find the shell-type dropdown (it currently shows **Bash**) and switch it to **PowerShell**. Cloud Shell restarts in a few seconds — you stay signed in as the same account, in the same subscription.

Confirm you landed in the same place:

```powershell
Get-AzContext
```

Now create a third resource group, this time with a PowerShell cmdlet:

```powershell
New-AzResourceGroup -Name "rg-catalog-ps-<yourinitials>" -Location "eastus" -Tag @{course="azure-core-solutions"; "created-by"="powershell"}
```

**Why bother with a third tool that does the same thing?** Two reasons. First, the outline lists PowerShell alongside the CLI as one of the ways teams automate Azure — Windows-heavy shops and teams already scripting in PowerShell for other Microsoft products often standardize on it instead of Bash, and you should recognize it on sight. Second, and more important for this lab: notice what did *not* change. You did not sign in again. You did not pick a subscription again. `Get-AzContext` returned the same subscription `az account show` did back in Step 5. That is Azure Resource Manager again — a third client, the same identity, the same control plane, the same rules.

**Why does the cmdlet look so different from the CLI command?** PowerShell cmdlets follow a strict **Verb-Noun** pattern — `New-AzResourceGroup`, `Get-AzResourceGroup`, `Remove-AzResourceGroup` — while the Azure CLI follows `az <noun> <verb>` — `az group create`, `az group show`, `az group delete`. Different grammar, same underlying request. If you can read one, you can guess the other: `Get-` is `show`/`list`, `New-` is `create`, `Remove-` is `delete`. Notice also that the tag argument is a PowerShell hashtable (`@{key="value"}`) instead of the CLI's `key=value` pairs — same data, different syntax for the same language's own conventions.

Switch back to **Bash** before continuing (the same dropdown) — the rest of this lab uses `az`.

---

## Step 10 – Verify All Three Groups from Every Door

Back in Bash, list everything:

```bash
az group list --output table
```

You should now see **three** resource groups — portal, CLI, and PowerShell — one list, three tools.

Now read the tags on the CLI-created group:

```bash
az group show --name rg-catalog-cli-<yourinitials> --query tags
```

Finally, go back to the portal, open **Resource groups**, and refresh. All three groups appear there too, including the one PowerShell created.

**Why check from three sides instead of two?** Because two could be a coincidence; three is a pattern. Portal, CLI, PowerShell — three different interfaces, three different people's preferred tool, one identical result each time. There is no "portal Azure," "CLI Azure," or "PowerShell Azure." There is one Azure, one control plane, and as many clients as anyone cares to write. Once this clicks, a whole category of confusion ("do I need to sync the portal with my script?") never happens to you.

**What is `--query` doing?** It filters the JSON output down to just the piece you asked for — here, the tags. The Azure CLI uses a query language called JMESPath for this. You do not need to master it today; just know that any time a command returns a wall of JSON, `--query` can extract exactly the field you care about.

> **Checkpoint:** All three resource groups appear in `az group list`, and each one carries the tags you gave it.

---

## Step 11 – Find Resources by Tag Across the Subscription

So far you have checked tags one resource group at a time. Real subscriptions have thousands of resources, and nobody reads them one at a time. In the portal, search for **Tags** and open it.

1. Click the **course** tag.
2. You should see every resource carrying that tag listed together — all three of your resource groups, regardless of which tool created them or what they are named.

**Why does this page matter more once you leave the classroom?** Because "find everything that belongs to the catalog project" or "find everything owner `js` is responsible for" are questions a real team asks constantly — during an audit, during an incident, during a cost review — and tags are the only mechanism that answers them *without* relying on naming conventions. A resource group named `rg-catalog-cli-js` tells a human what it is, but a script or a report has to parse that name and guess. A tag is structured data built for exactly this kind of query. This is the same mechanism Microsoft Cost Management and Azure Policy lean on later in the course — you are looking at its simplest form today.

> **Checkpoint:** The Tags page shows all three of your resource groups under `course`.

---

## Step 12 – Read the Activity Log: ARM's Receipt

In the portal, open `rg-catalog-cli-<yourinitials>` (the one you created from the command line), then open **Activity log** in its left menu.

You should see an **Update resource group** operation, recorded a few minutes ago, with your account as the caller — even though this group was created from Cloud Shell, not the portal.

> **Why "Update" and not "Create"?** ARM records creating and updating a resource group as the same *write* operation, and the portal labels that operation "Update resource group." The entry you are looking at *is* your create.

**Why is this the most interesting page in the lab?** Because it is Azure Resource Manager's receipt. Every create, update, and delete — from the portal, the CLI, PowerShell, or a program — passes through ARM, and ARM logs every one of them in the same place. The portal is simply *displaying* that log. This is what "one control plane" buys you in practice: one audit trail, no matter which tool anyone on the team prefers. When something changes unexpectedly in a real subscription, this log is where the investigation starts.

Now open `rg-catalog-ps-<yourinitials>` — the group you created with PowerShell — and check its Activity log too. Click into the entry and expand it.

**What is different about this entry, and what is the same?** The event's details will show the request came from a different client (Azure PowerShell, rather than the Azure CLI), and possibly a different-looking correlation ID. That is the "different" part — ARM does track which tool made each call, which is exactly how a real investigation narrows down "who ran this and from where." The "same" part is everything that matters for governance: the operation name, the outcome, and the fact that it is sitting in the identical Activity log view as your CLI and portal groups. Three tools, three slightly different fingerprints, one uniform record.

---

## Step 13 – Clean Up

Deleting a resource group deletes everything inside it. All three of yours are empty, so this is safe and quick.

In Cloud Shell:

```bash
az group delete --name rg-catalog-portal-<yourinitials> --yes --no-wait
az group delete --name rg-catalog-cli-<yourinitials> --yes --no-wait
az group delete --name rg-catalog-ps-<yourinitials> --yes --no-wait
```

**Why does one CLI command clean up the group PowerShell created?** Because delete, like create, is just a request to Azure Resource Manager — the tool that issued the original request has no special claim over undoing it. `Remove-AzResourceGroup -Name "rg-catalog-ps-<yourinitials>"` would work identically if you switched back to PowerShell; using the CLI here simply saves you a shell swap. This is the same "one control plane" idea from Step 9, viewed from the exit instead of the entrance.

Confirm they are going (deletion takes a minute or two):

```bash
az group list --output table
```

**Why is cleanup a step and not an afterthought?** This is the single most important habit in cloud work. Consumption pricing means anything left allocated keeps billing, and *the provider will not clean up after you* — cleanup sits on the customer side of the shared responsibility line. Today it is pure tidiness because resource groups are free. But you are building the reflex for later today, when the resources inside will not be free. A resource group delete is the fastest way to guarantee nothing is left behind — which is itself a reason to keep resources that live and die together in the same group.

**What does `--no-wait` do?** It tells the CLI to submit the delete and return immediately instead of blocking until the deletion finishes. The deletion continues in the background — ARM does the work, not your terminal.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Subscription** | The billing and quota boundary; the Cost analysis page reads its meter |
| **Resource group** | A free, logical container for resources that share a lifecycle |
| **Resource group region** | Sets where the group's *metadata* lives, not where its resources must run |
| **Cloud Shell** | A zero-setup terminal in the browser, already signed in as you |
| **`az group create` / `list` / `show` / `delete`** | The CLI verbs you will use most, with `--output` and `--query` to shape results |
| **Azure PowerShell** | The `Verb-Noun` cmdlet style (`New-AzResourceGroup`), a third client of the same control plane |
| **Tags** | Labels applied at creation time that make cost and ownership traceable |
| **Tags page** | Finds every resource with a given tag across the whole subscription, regardless of name or tool |
| **Budgets** | A guardrail set *before* spending happens, not a lock — it alerts, it does not block |
| **Azure Resource Manager** | One control plane behind every tool — proven by identical results and a single activity log |
| **Cleanup** | The customer-side habit that prevents surprise bills |

---

## Reflection

Answer for yourself or discuss with a partner:

1. Why did the portal, the CLI, and PowerShell all produce the same kind of resource group?
2. Your resource group was in East US. Could you have placed a virtual machine from West Europe inside it? Why or why not?
3. What is the risk of skipping the cleanup step once you start creating resources that bill?
4. `New-AzResourceGroup` and `az group create` look nothing alike on the page. Why doesn't that difference matter to Azure Resource Manager?
5. A budget alert emailed the team that spending crossed 80% of its threshold. Nothing stopped running. Why is that the correct behavior for a budget, and what would you reach for if you actually wanted spending to stop?

---

## Optional Challenge

Create a fourth resource group in a **different region** than your first, with two tags applied in one command (`course` and `owner`), using whichever tool (CLI or PowerShell) you did not use last. Confirm the region and tags using `az group show` (or `Get-AzResourceGroup`) with a query, then delete it. If you finish early, run `az group list --output json` and compare it to the table output — find three properties the table view was hiding from you.

---

## Conclusion

You signed in to a brand-new subscription, learned the four landmarks that prevent most beginner mistakes, found the page where spending is tracked, and created the same resource three ways — portal, CLI, and PowerShell — then watched one activity log and one tag search treat all three identically. The portal, the CLI, and PowerShell are different doors into one control plane, and you have now walked through all three.

In the next lab you will use that control plane against itself: you will lock a resource and watch Azure Resource Manager refuse to delete it — from every door at once.
