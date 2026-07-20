# One App, Two Ways – Deploying with IaaS and PaaS

## The Scenario

The groundwork at **Cascade Outfitters** is done: you know your way around, and the project has a home. Now the actual work arrives. The catalog team has a simple product page ready, and your lead wants a recommendation backed by evidence, not slideware:

> "Deploy the catalog page twice — once on a virtual machine, once on App Service. Same page, two service models. Then tell me what we'd actually be signing up to manage in each case. The team is three people; every hour spent patching servers is an hour not spent on the catalog."

This lab is that bake-off. You will build a working web server from a raw virtual machine (IaaS), then deploy the identical content to Azure App Service (PaaS) — and feel the difference in your hands: what you had to do, what you had to know, and what you would own at 2 a.m. when something breaks.

By the end, the IaaS/PaaS/SaaS spectrum from the slides stops being a diagram and becomes a decision you can defend.

Then you will go one step further than IaaS and PaaS: a third deployment on **Azure Functions**, the serverless end of the spectrum, so "pay per execution" stops being a slide bullet and becomes a bill you can actually reason about.

> **Cost note:** This lab creates resources that bill — a small VM costs a few cents per hour, and the Functions deployment needs a small storage account that costs a fraction of a cent. The App Service and Functions consumption tiers we use are both free at this scale. Everything is deleted in the cleanup step, and skipping cleanup is the only way this lab costs more than pocket change. Do not skip cleanup.

### Prerequisites

- Completion of the previous labs, or comfort with Cloud Shell and resource groups
- An Azure account (the free account is enough)

---

## Step 1 – Create the Resource Group for the Bake-Off

In Cloud Shell (Bash), run (use your initials):

```bash
az group create \
  --name rg-compute-<yourinitials> \
  --location centralus \
  --tags project=catalog environment=dev owner=<yourinitials>
```

**Why one group for both deployments?** Both the VM and the App Service exist for the same purpose — this comparison — and they will be deleted together when it ends. Resources that share a lifecycle share a resource group; that way cleanup is one command instead of a scavenger hunt. The tags are now habit, not homework.

---

## Step 2 – Write the Server's Setup Instructions Before the Server Exists

Create a file that describes what the VM should do to itself on first boot. In Cloud Shell:

```bash
mkdir -p ~/compute-lab && cd ~/compute-lab

cat > cloud-init.yaml << 'EOF'
#cloud-config
package_update: true
packages:
  - nginx
write_files:
  - path: /var/www/html/index.html
    content: |
      <html>
        <head><title>Cascade Outfitters Catalog</title></head>
        <body style="font-family: sans-serif; margin: 3em;">
          <h1>Cascade Outfitters - Product Catalog</h1>
          <p>Served from an <strong>Azure Virtual Machine</strong> (IaaS).</p>
          <p>Someone on our team patches this OS. That someone is us.</p>
        </body>
      </html>
runcmd:
  - systemctl enable --now nginx
EOF
```

**What is cloud-init?** A standard first-boot configuration format that most Linux cloud images understand. When Azure creates the VM, it hands this file to the OS, which then updates its package list, installs nginx, writes your web page, and starts the service — all before you ever log in.

**Why script the setup instead of logging in and typing?** Two reasons, and both matter beyond this lab. First, **repeatability**: this file produces the same server every time, with no missed step and no "it works on the one I set up by hand." Second, it demonstrates where IaaS responsibility begins — notice that *you* are the one specifying packages, web content, and service startup. Azure gives you a clean OS and walks away. Everything above the hypervisor is your job, and this file is you doing that job. Keep that thought; it is the entire point of the comparison to come.

---

## Step 3 – Create the Virtual Machine

```bash
az vm create \
  --resource-group rg-compute-<yourinitials> \
  --name vm-catalog \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init.yaml
```

This takes two to four minutes. While it runs, read the flags — every one is a decision you just made:

| Flag | The decision | Why it matters |
|---|---|---|
| `--image Ubuntu2204` | Which operating system | In IaaS, the OS is yours: yours to choose, yours to patch, yours to upgrade when it goes end-of-life |
| `--size Standard_D2s_v3` | How much CPU and memory | A small general-purpose size (2 vCPUs) — right for a lab, wrong for a busy production site. Sizing is your job in IaaS |
| `--admin-username` / `--generate-ssh-keys` | How you log in | Keys are generated if you have none, and the public key is placed on the VM. Password logins are weaker; keys are the norm |
| `--custom-data cloud-init.yaml` | What the server becomes | Your Step 2 file, injected at first boot |

> **Note:** If the command reports the size is unavailable or quota is exceeded, ask your instructor before retrying — the class uses Central US deliberately, and switching regions on your own can land you somewhere the rest of the lab has not been tested. A failed VM create often leaves partial networking resources behind, so any retry starts by deleting the resource group (`az group delete --name rg-compute-<yourinitials> --yes`) and recreating it.

**What did Azure create besides the VM?** When it finishes, run `az resource list --resource-group rg-compute-<yourinitials> --output table`. You will see roughly six resources: the VM, a disk, a network interface, a public IP address, a network security group — and a **virtual network** the CLI quietly created to put it all in. A "virtual machine" is really a small bundle: compute, storage, and networking are separate resources wired together. Today the CLI chose sensible defaults for that wiring; the networking lab takes direct control of it, starting with that quietly-created network.

---

## Step 4 – Open the Firewall and Visit Your Server

By default, nothing can reach the VM from the internet on port 80. Open it:

```bash
az vm open-port --resource-group rg-compute-<yourinitials> --name vm-catalog --port 80
```

Get the public IP address:

```bash
az vm show --resource-group rg-compute-<yourinitials> --name vm-catalog \
  --show-details --query publicIps --output tsv
```

Now test it — either paste the IP into your browser, or from Cloud Shell:

```bash
curl http://<the-ip-address>
```

You should see the catalog page HTML, served by *your* nginx on *your* VM.

> **If the connection is refused:** cloud-init runs *after* the VM boots, and the package update plus nginx install can take an extra minute or two beyond `vm create` returning. Wait a moment and retry — watching a server finish assembling itself is part of the IaaS experience.

**Why was port 80 closed to begin with?** Azure's default posture for a new VM is deny-by-default for inbound traffic — only what you explicitly allow gets in. That is the right default: the internet continuously scans every public IP address, and an accidentally exposed service is one of the most common cloud security incidents. `az vm open-port` added an allow rule for port 80 to the VM's network security group. Notice this is *your* security decision to make — in IaaS, the responsibility line puts network configuration on your side.

> **Checkpoint:** The catalog page loads from the VM's public IP.

---

## Step 5 – Log In and Meet Everything You Now Own

SSH into the VM from Cloud Shell:

```bash
ssh azureuser@<the-ip-address>
```

(Answer `yes` to the fingerprint prompt.) Once inside, look around:

```bash
systemctl status nginx --no-pager     # the web server you now operate
apt list --upgradable 2>/dev/null | head    # patches waiting for a decision
df -h /                               # a disk that can fill up
exit
```

**Why poke around a working server?** Because this is the honest face of IaaS. That `apt list --upgradable` output is a to-do list with your name on it — every one of those packages, the OS itself, nginx, its configuration, TLS certificates, log rotation, disk space, and backups are the customer's side of the shared responsibility model. None of it is hard. All of it is *recurring*. For a three-person team, the question is never "can we run a VM?" — it is "is running a VM the best use of our hours?" Hold that question through the next two steps.

**When is all this ownership worth it?** When you *need* what it buys: full OS control, custom system software, unusual runtimes, lift-and-shift of an app that expects a server. IaaS is not the wrong choice — it is the maximum-control, maximum-responsibility choice.

---

## Step 6 – Scale the VM Vertically

The slides drew a line between two kinds of scaling: **vertical** (a bigger machine) and **horizontal** (more machines). You have a machine sitting right here — resize it.

```bash
az vm deallocate --resource-group rg-compute-<yourinitials> --name vm-catalog
az vm resize --resource-group rg-compute-<yourinitials> --name vm-catalog --size Standard_D4s_v3
az vm start --resource-group rg-compute-<yourinitials> --name vm-catalog
```

Confirm the new size:

```bash
az vm show --resource-group rg-compute-<yourinitials> --name vm-catalog --query hardwareProfile.vmSize --output tsv
```

**Why deallocate before resizing?** A running VM occupies specific physical hardware sized to fit its current CPU and memory footprint. Resizing to `Standard_D4s_v3` (double the vCPUs and memory of your `Standard_D2s_v3`) may need to move the VM to different hardware, and Azure will not do that out from under a running OS. `deallocate` releases the VM (and stops billing for compute — only the disk keeps its small storage charge while stopped), `resize` changes the specification ARM stores for it, and `start` boots it again, this time provisioned against the new size. The whole cycle takes a minute or two.

**Why is this "vertical scaling," and why does the slide deck warn it is not the whole answer?** You just made *this one machine* bigger — more CPU, more memory, same single point of failure. It is the fastest way to buy headroom, and it is also a dead end: there is a largest VM size, and even before you reach it, a single bigger machine still goes down for the minutes it takes to resize, as you just watched with `deallocate`/`start`. **Horizontal** scaling — more machines behind a load balancer — is what actually gets you *elasticity* (capacity that grows and shrinks with demand, no restart required) and survives a single machine failing. You will build the horizontal pattern with a load balancer in the networking lab. Today, notice the trade you just made: `az vm resize` is one command, but it bought you a bigger single point of failure, not a more resilient one.

Curl the site once more to confirm nothing else changed:

```bash
curl http://<the-ip-address>
```

**Checkpoint:** `hardwareProfile.vmSize` reports `Standard_D4s_v3`, and the catalog page still loads — same server, same content, more room underneath it.

---

## Step 7 – Now the PaaS Way: Prepare the Same Content

Back in Cloud Shell, create the same page as a plain folder of content:

```bash
mkdir -p ~/compute-lab/catalog-site && cd ~/compute-lab/catalog-site

cat > index.html << 'EOF'
<html>
  <head><title>Cascade Outfitters Catalog</title></head>
  <body style="font-family: sans-serif; margin: 3em;">
    <h1>Cascade Outfitters - Product Catalog</h1>
    <p>Served from <strong>Azure App Service</strong> (PaaS).</p>
    <p>Nobody on our team patches this OS. That is the point.</p>
  </body>
</html>
EOF
```

**Notice what is missing.** There is no cloud-init file. No package list. No nginx. No systemctl. Just the content itself. In PaaS, the platform brings its own web server, its own OS, and its own patching schedule — you bring the application. The next command will make that concrete.

---

## Step 8 – Deploy to App Service with One Command

App Service names become part of a public URL, so yours must be globally unique — include your initials and a few digits:

```bash
az webapp up \
  --name catalog-<yourinitials>-<3digits> \
  --resource-group rg-compute-<yourinitials> \
  --plan plan-catalog-<yourinitials> \
  --location centralus \
  --sku F1 \
  --html
```

In about a minute, the output prints a URL like `http://catalog-js-042.azurewebsites.net`. Open it in your browser.

**What just happened?** One command created an **App Service plan** (the pool of compute your apps run on), created the **web app**, zipped your folder, uploaded it, and served it — with a real DNS name and HTTPS available out of the box. Compare that to Steps 2 through 4.

**Why is there still a "plan" with a size (`--sku F1`)?** Because PaaS abstracts the *operating system*, not the laws of physics. Your app still runs on compute somewhere, and the plan is how you choose and pay for it. `F1` is the free tier — fine for this lab, limited for real use. The difference from the VM is *what the size decision includes*: with the plan you choose capacity, but the OS, web server, patching, and runtime upkeep stay on Microsoft's side of the line.

**What did you give up?** Honesty matters in a bake-off. You cannot SSH into an F1 App Service and install a system package. You cannot run arbitrary daemons or pick the kernel. If the catalog app someday needs an exotic native library or a custom network stack, PaaS may refuse — and that is exactly when IaaS earns its keep. Control and convenience trade off; that *is* the service-model spectrum.

> **Checkpoint:** Both pages are live — one at the VM's IP address, one at the `azurewebsites.net` URL — and each page tells you which model served it.

---

## Step 9 – Prepare a Serverless Function

There is one more stop on the spectrum, further than App Service: **Azure Functions**. Instead of a server that runs continuously waiting for requests, a function exists only while it is handling one, and you are billed per execution rather than per hour. Cloud Shell's Bash environment includes the Functions Core Tools (`func`) already installed, so you can build one the same way you would on a laptop.

> **First, confirm the tool is present:** run `func --version`. If it prints a version starting with `4`, you are set. If the command is not found, install it with `npm install -g azure-functions-core-tools@4 --unsafe-perm true` and re-run `func --version`.

```bash
cd ~/compute-lab
func init catalog-function --javascript
cd catalog-function
func new --name CatalogLookup --template "HTTP trigger" --authlevel anonymous
```

Open the generated function and see what it contains:

```bash
cat src/functions/CatalogLookup.js
```

> **Note:** Current Core Tools scaffold the **v4 Node programming model**, which puts each function in `src/functions/<name>.js` and defines its trigger in code — there is no per-function folder and no `function.json`. (Older Core Tools used the v3 layout, `CatalogLookup/index.js` with a separate `function.json`.) If the `cat` above says "No such file," run `find . -name '*.js' -not -path '*/node_modules/*'` to see where your version put it.

**What did `func init` and `func new` just do?** `func init` scaffolded a function app project — a small folder structure Azure Functions knows how to run, comparable to what `az webapp up` deployed for you invisibly in Step 8. `func new` added one function to it, triggered by an HTTP request, and wrote a working handler that reads a `name` query parameter and returns a greeting. Notice there is no web server in this code, no listening socket, no port binding — the trigger and the runtime that answers HTTP requests are the platform's job, not yours. You wrote a function; Azure Functions supplies everything around it.

**Why does the *trigger* matter more than the code?** Because the trigger is the entire idea of serverless in one word: this function runs only when an HTTP request arrives (or, in other apps, when a queue message lands, a timer fires, or a file is uploaded). Between triggers, nothing is running and nothing is billing. Compare that to your VM, which billed every hour whether or not a single visitor showed up, or your App Service plan, which reserves capacity continuously. Functions is the one deployment in this lab where "idle" and "free" are the same thing.

---

## Step 10 – Deploy and Test the Function

A function app needs a storage account behind it (to manage triggers and logs) and a function app resource to run in:

```bash
az storage account create \
  --name stfunc<yourinitials> \
  --resource-group rg-compute-<yourinitials> \
  --location centralus \
  --sku Standard_LRS

az functionapp create \
  --resource-group rg-compute-<yourinitials> \
  --consumption-plan-location centralus \
  --runtime node \
  --runtime-version 24 \
  --functions-version 4 \
  --name func-catalog-<yourinitials> \
  --storage-account stfunc<yourinitials>
```

> **Note:** Storage account names must be globally unique, lowercase, 3–24 characters, letters and digits only. Add digits if `stfunc<yourinitials>` is taken.

Now publish your code to it:

```bash
func azure functionapp publish func-catalog-<yourinitials>
```

The command prints an **Invoke URL** when it finishes — for an anonymous function it looks like `https://func-catalog-<yourinitials>.azurewebsites.net/api/cataloglookup`, with no query string on the end. Test it by adding a `name` query parameter with **`?`** (a query string always starts with `?`):

```bash
curl "<the-invoke-url>?name=Catalog"
```

You should get back a greeting containing the name you passed — with the v4 model template it is `Hello, Catalog!` (older templates return a longer sentence like `Hello, Catalog. This HTTP triggered function executed successfully.`). Either way, seeing your `name` reflected back confirms the function ran.

> **Note:** Start the parameter with `?`, not `&`. Use `&` only to add a *second* parameter to a URL that already has a `?`. If you created the function with function-level auth instead of `anonymous`, the invoke URL would already end in `?code=...`, and only then would you append `&name=Catalog`.

**Why did this deployment need a storage account when App Service did not?** Because Functions' billing and scaling model depends on tracking function state and logs outside of any single running instance — there is no "instance" sitting idle between requests the way there is with a VM or an App Service worker. The storage account is where that bookkeeping lives. It is a small, structural cost of getting the "billed only while running" model, not a design choice you can skip.

**Why does `--consumption-plan-location` matter here specifically?** The **Consumption plan** is what makes this serverless: Azure allocates compute for your function on demand, for the duration of each execution, and then releases it. Contrast this with the App Service plan from Step 8, which reserved capacity continuously whether a request was in flight or not. If Cascade Outfitters' catalog lookup gets hit once a minute, Functions on Consumption is close to free; the same traffic pattern on a reserved VM or App Service plan pays for 24 hours of standby every day.

> **Checkpoint:** Curling the Invoke URL returns a response, and no server, port, or firewall rule was ever mentioned to make that happen.

---

## Step 11 – Compare What You Actually Did

Look back at your terminal history and fill in the scorecard your lead asked for:

| | VM (IaaS) | App Service (PaaS) | Functions (serverless) |
|---|---|---|---|
| Steps to get to a working endpoint | Write cloud-init, create VM, open port, find IP | One `az webapp up` | `func init`/`func new`, create storage + function app, `func publish` |
| Who chose and patches the OS | **You** | Microsoft | Microsoft — there is no OS you can see |
| Who operates the web server (nginx/IIS) | **You** | Microsoft | N/A — no server process runs between requests |
| Who configures inbound network access | **You** (the port was closed until you opened it) | Platform (HTTP/HTTPS just work) | Platform |
| Billed for | Every hour the VM is allocated, busy or not | Every hour the plan is provisioned, busy or not | Only the seconds each function actually executes |
| Can install anything, control everything | **Yes** | No — platform constraints apply | No — code only, shortest execution window |
| What you deploy | A whole configured machine | Application content | A single function's code |
| Recurring operational load | Patching, upgrades, disk, backups, hardening | Your app and its code | Your code only — no idle capacity to manage at all |

**So what do you tell your lead?** For a three-person team shipping a standard web app, PaaS wins the main site: the entire row of recurring server chores disappears, and the hours go to the catalog. Functions wins for anything event-driven and bursty — a background job that runs a few times an hour has no business reserving a server plan around the clock. The VM path is the right answer when the workload demands OS-level control or is being lifted from a datacenter as-is. The model is a choice about *where to spend responsibility and money*, not about which technology is better.

---

## Step 12 – Place the Rest of the Compute Menu

You deployed three points on the spectrum yourself, but the Azure compute menu is longer still, and your lead will hear all of these names in vendor meetings. Every one of them is a different answer to the same two questions your scorecard just answered: *how much control do you need, and what are you staffed to operate?*

| Service | What it is | Reach for it when |
|---|---|---|
| **Virtual Machines** | The IaaS you just built | You need OS control, custom system software, or a lift-and-shift |
| **VM Scale Sets** | A fleet of identical VMs that grows and shrinks automatically | Your VM workload needs to handle variable load — this is the *horizontal* scaling from Step 6 done automatically. Your single `vm-catalog` handled one visitor fine; a scale set is how the same design survives a holiday sale |
| **Azure Virtual Desktop** | Full Windows desktops delivered from Azure | Users need a managed desktop from anywhere — remote work, contractors, secure workstations. It is IaaS solving an end-user problem, not an app-hosting one |
| **App Service** | The PaaS you just used | Standard web apps and APIs — the default choice for teams like the catalog team |
| **Azure Functions** | The serverless deployment you just built | Glue code and background jobs: "when an order lands in the queue, resize the product image." No app is running — or billing — between events |
| **Container Instances (ACI)** | Run a single container, no cluster, per-second billing | A quick job or task that ships as a container |
| **Container Apps** | Containers with scaling and HTTPS handled for you | Containerized apps and microservices without cluster management — roughly "App Service for containers" |
| **Kubernetes Service (AKS)** | A managed Kubernetes cluster you operate | Many containers, complex orchestration needs, and — critically — engineers dedicated to running the platform |

**Why do containers get three separate services?** Because a container answers a packaging question — "how do I ship my app with its dependencies so it runs the same everywhere?" — but not an operations question. ACI, Container Apps, and AKS are the same trade you spent this lab measuring, replayed for containers: minimum effort, managed middle, maximum control. If you can defend your VM-versus-App-Service-versus-Functions recommendation, you can defend an AKS-versus-Container-Apps one with the same reasoning.

**The honest summary for a three-person team:** App Service for the catalog site, Functions for its background jobs, and nobody touches AKS until there is a platform team to feed it. That sentence — matching the service to both the workload *and* the staffing — is the skill this module is actually teaching.

---

## Step 13 – Clean Up (Not Optional Today)

> **Trying the Optional Challenge below? Do it first.** It reuses the App Service app (and Function) you just built, so run the challenge *before* this cleanup — once the resource group is deleted, those resources are gone.

This is the first lab where skipping cleanup costs real money — the VM bills every hour it exists, page or no page. Everything lives in one resource group, so one command ends it:

```bash
az group delete --name rg-compute-<yourinitials> --yes --no-wait
```

Verify it is going:

```bash
az group list --output table
```

**Why does deleting the group beat deleting each resource one at a time?** Remember Step 3: the "VM" was actually six resources, and Step 10 added a function app plus its own storage account on top of that. Deleting only the VM would leave the disk, the public IP, the Functions storage account, and the rest behind — some of which keep billing quietly. Because everything from this lab shares one resource group, regardless of whether it was a VM, a web app, or a serverless function, the group delete guarantees nothing is orphaned. This is the lifecycle-container idea paying rent: cleanup was designed in back at Step 1, not improvised at the end.

> **Checkpoint:** `rg-compute-<yourinitials>` no longer appears in the list (deletion may take a few minutes to finish).

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **IaaS (Azure VM)** | Full control: you chose the OS, installed the server, opened the port — and inherited the patching to-do list |
| **cloud-init / custom data** | First-boot automation; repeatable server setup instead of hand-typed steps |
| **A VM is a bundle** | Compute, disk, NIC, public IP, and NSG are separate resources wired together |
| **Deny-by-default networking** | Port 80 was closed until you explicitly opened it — inbound exposure is your decision in IaaS |
| **Vertical scaling** | `az vm resize` buys a bigger single machine — fast, but still one point of failure |
| **PaaS (App Service)** | You brought content; the platform brought OS, web server, DNS, and TLS |
| **App Service plan** | PaaS still runs on compute you choose and pay for — it abstracts the OS, not capacity |
| **Serverless (Functions)** | No server at all between requests — billed per execution, on a Consumption plan that allocates compute on demand |
| **Triggers** | The event (HTTP, queue, timer) that wakes a function; nothing runs — or bills — between triggers |
| **The trade** | Control, convenience, and billing model all move together along the IaaS→PaaS→serverless spectrum |
| **The full compute menu** | VMs, Scale Sets, Virtual Desktop, App Service, Functions, ACI, Container Apps, AKS — the same control-vs-convenience decision at eight price points |
| **Lifecycle cleanup** | One resource group per purpose makes teardown one command with nothing orphaned |

---

## Reflection

Answer for yourself or discuss with a partner:

1. The same page needed a five-flag `vm create` plus a cloud-init file on IaaS, and one `webapp up` on PaaS. Where did all that work *go* — who is doing it now?
2. Name a workload for which the VM would have been the right answer despite the extra responsibility.
3. On the shared responsibility diagram from the slides, put a finger on the line for your VM deployment, then for your App Service deployment, then for your Function. Which rows moved each time?
4. Your VM's public IP responded on port 80 but nothing else. Why is that the correct default, and who chose to open 80?
5. Resizing the VM in Step 6 took a deallocate-resize-start cycle and a few minutes of downtime. What would happen to that downtime if, instead of a bigger VM, you had put a *second* VM behind a load balancer? Why is that a different kind of scaling, not just a bigger version of the same one?
6. Your Function billed nothing while it wasn't handling a request. Name one Cascade Outfitters workload from this course's scenario that fits that billing model well, and one that would not.

---

## Optional Challenge

> **Do these before the Clean Up step (Step 13).** Both reuse resources from this lab — your App Service app and your Function — so they must still exist. If you already deleted the resource group, you would have to redeploy from Steps 7–10 first.

Two independent challenges, pick either or both:

1. Redeploy the App Service page with a visible change: edit `index.html` (add a product list), then run the same `az webapp up` command again from the `catalog-site` folder and refresh the browser. Time it. Then estimate: what would the same content change have required on the VM path? This gap — seconds versus a login-edit-verify cycle — is why teams that ship frequently gravitate toward PaaS and automation.

2. Edit `src/functions/CatalogLookup.js` in the Functions project to change the response message, then run `func azure functionapp publish func-catalog-<yourinitials>` again and re-test the Invoke URL. Time this redeploy too, and compare it against both of the others. Then check the **Monitor** tab on your function in the portal (search **Function App**, open `func-catalog-<yourinitials>`, open `CatalogLookup`, then **Monitor**) — you should see one logged execution per request you made. This is the billing unit for serverless made visible: not server-hours, but individual invocations.

(If you attempt either, remember cleanup afterward.)

---

## Conclusion

You deployed the same idea three ways and lived the differences. On the VM you were the systems administrator: you specified the OS setup, opened the firewall, resized the hardware under a running service, and logged in to a machine whose patch list now has your name on it. On App Service you were only the developer: content in, URL out, and the operating system became someone else's problem. On Functions you did not even think about a running server — you wrote a handler for one trigger, and Azure metered you by the execution instead of the hour.

That trade — control and continuous cost on one end, convenience and pay-per-use on the other — is the single decision you will make most often in cloud architecture. In the next lab you take back one layer deliberately: the network. You will build the virtual network, subnets, and security rules that production workloads sit inside — and prove with packets who can reach what.
