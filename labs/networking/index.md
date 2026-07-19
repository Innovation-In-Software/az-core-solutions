# Build a Segmented Network – VNets, Subnets, and NSGs

## The Scenario

The compute bake-off convinced **Cascade Outfitters** — but it also raised a flag in the architecture review. In the last lab, the CLI quietly created a network around your VM with sensible defaults. The security lead was not thrilled by the word *quietly*:

> "Before this goes anywhere near production, I want the network designed on purpose. Web servers reachable from the internet on exactly the ports we choose. The database tier reachable *only* from the web tier — no public address at all. And I don't want a diagram that claims it works; I want you to prove it with packets."

This lab builds that design from scratch. You will create a virtual network with two subnets — a **web tier** and a **data tier** — place a VM in each, and control every path with **network security groups (NSGs)**. Then you will run the proofs: show the web server answering the internet, show the data server unreachable from the internet, show the web tier reaching the data tier — and then break and restore that path by changing one rule, watching NSG priority decide the winner.

This is the pattern behind almost every real deployment in Azure. Learn it once with two VMs and you will recognize it in systems with two hundred.

> **Cost note:** This lab runs three small VMs (two web, one data), a few cents per hour total. Step 10 also adds a Standard-SKU load balancer and a Standard-SKU public IP, which each carry a small hourly charge (unlike the free Basic tier used elsewhere) — still pennies for the length of a class, but the most expensive per-hour items in these four labs. Everything is deleted in the cleanup step. Do not skip cleanup.

### Prerequisites

- Completion of the previous labs, or comfort with Cloud Shell, resource groups, and `az vm create`
- An Azure account (the free account is enough)

---

## Step 1 – Sketch Before You Build

Here is the target. Keep it visible — every command that follows creates one piece of it.

```
Internet
   │
   │  allowed: port 80 (HTTP), port 22 (SSH, for the lab)
   ▼
┌─────────────────────────────────────────────────────┐
│  vnet-catalog          10.20.0.0/16                 │
│                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐ │
│  │ snet-web             │  │ snet-data            │ │
│  │ 10.20.1.0/24         │  │ 10.20.2.0/24         │ │
│  │ [nsg-web]            │  │ [nsg-data]           │ │
│  │                      │  │                      │ │
│  │  vm-web              │──▶  vm-data             │ │
│  │  (public IP)         │  │  (NO public IP)      │ │
│  └──────────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Why a /16 for the VNet and /24s for the subnets?** The VNet's address space (`10.20.0.0/16` — about 65,000 addresses) is the total private range this network owns; each subnet carves out a /24 slice (about 250 usable addresses). This is deliberately roomy: address spaces are painful to change later, and adding a *third* subnet tomorrow (say, a management tier) should be a carve-out, not a redesign. Plan the space bigger than the project.

**Why two subnets instead of putting both VMs in one?** Because a subnet is the natural unit of network policy. Security rules attach cleanly at the subnet boundary, so "the web tier may talk to the data tier on one port" becomes a rule between two subnets rather than bookkeeping about individual machines. When the web tier grows from one VM to ten, the rules already cover them.

---

## Step 2 – Create the Resource Group and Virtual Network

```bash
az group create \
  --name rg-network-<yourinitials> \
  --location centralus \
  --tags project=catalog environment=dev owner=<yourinitials>
```

Now the network, with the web subnet in the same command:

```bash
az network vnet create \
  --resource-group rg-network-<yourinitials> \
  --name vnet-catalog \
  --address-prefix 10.20.0.0/16 \
  --subnet-name snet-web \
  --subnet-prefix 10.20.1.0/24
```

Add the data subnet:

```bash
az network vnet subnet create \
  --resource-group rg-network-<yourinitials> \
  --vnet-name vnet-catalog \
  --name snet-data \
  --address-prefix 10.20.2.0/24
```

**What exactly is a VNet?** Your own private slice of Azure's network: an isolated address space where *you* decide the layout and the traffic rules. Resources inside it reach each other over private IPs by default and are unreachable from the internet unless you explicitly arrange otherwise. It is the software equivalent of the switches and VLANs a datacenter team used to rack — created in seconds, defined entirely in configuration.

**Why do the private addresses start with 10.?** The `10.0.0.0/8` range (along with `172.16/12` and `192.168/16`) is reserved worldwide for private networks — never routed on the public internet. Every organization can use them, which is why your VNet can hand out `10.20.1.x` addresses without asking anyone. It also means a private IP alone can never make a VM internet-reachable — that will matter in the proofs later.

---

## Step 3 – Create the NSGs and the Web Tier Rules

Create one network security group per tier:

```bash
az network nsg create --resource-group rg-network-<yourinitials> --name nsg-web
az network nsg create --resource-group rg-network-<yourinitials> --name nsg-data
```

Give the web NSG its two inbound allowances:

```bash
az network nsg rule create \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-web \
  --name allow-http \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-port-ranges 80

az network nsg rule create \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-web \
  --name allow-ssh \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-port-ranges 22
```

**What is an NSG, really?** A stateful firewall expressed as a list of rules. Each rule matches traffic by direction, source, destination, port, and protocol, and either allows or denies it. *Stateful* means return traffic is automatic: when a browser's request is allowed in on port 80, the response is allowed back out without a separate rule.

**Read one rule as a sentence.** The first one says: *inbound TCP traffic from any internet address to port 80 is allowed, and this rule wins against anything with a priority number above 100.* Every NSG rule is a sentence of that shape:

| Field | In `allow-http` | Meaning |
|---|---|---|
| `direction` | Inbound | Applies to traffic arriving, not leaving |
| `source-address-prefixes` | `Internet` | A service tag meaning "any public address" — Azure maintains these named groups so you don't hand-manage IP lists |
| `destination-port-ranges` | 80 | The port the traffic is heading for |
| `access` | Allow | Permit the match |
| `priority` | 100 | Lower number = evaluated first; first match wins |

**Why allow SSH from the whole internet?** For lab convenience only — Cloud Shell's outbound address varies, so pinning the source is fiddly in a classroom. In production you would restrict the source to a known range, or better, use Azure Bastion and allow no public SSH at all. Note the deliberate contrast: `nsg-data` is getting no internet rules of any kind.

---

## Step 4 – Attach the NSGs to the Subnets

```bash
az network vnet subnet update \
  --resource-group rg-network-<yourinitials> \
  --vnet-name vnet-catalog \
  --name snet-web \
  --network-security-group nsg-web

az network vnet subnet update \
  --resource-group rg-network-<yourinitials> \
  --vnet-name vnet-catalog \
  --name snet-data \
  --network-security-group nsg-data
```

**Why attach at the subnet instead of the VM?** An NSG can attach to a subnet (governing everything in it) or to an individual VM's network interface (governing just that machine). Subnet attachment is the scalable habit: every VM that ever lands in `snet-web` — including the nine the team adds next quarter — inherits the tier's rules automatically, with nothing to remember per machine. NIC-level NSGs exist for genuine per-machine exceptions; when both are present, traffic must pass **both**.

**Wait — `nsg-data` has no rules. What does an empty NSG do?** More than you might think, and Step 7 will show you exactly what. Hold the question.

---

## Step 5 – Create the Two VMs

Reuse the cloud-init trick from the compute lab so the web server configures itself. First the file:

```bash
mkdir -p ~/network-lab && cd ~/network-lab

cat > cloud-init-web.yaml << 'EOF'
#cloud-config
package_update: true
packages:
  - nginx
write_files:
  - path: /var/www/html/index.html
    content: |
      <html><body style="font-family: sans-serif; margin: 3em;">
        <h1>Cascade Outfitters – Web Tier</h1>
        <p>Reachable from the internet on port 80. Nothing else is.</p>
      </body></html>
runcmd:
  - systemctl enable --now nginx
EOF
```

The web VM — public IP, in `snet-web`:

```bash
az vm create \
  --resource-group rg-network-<yourinitials> \
  --name vm-web \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --vnet-name vnet-catalog \
  --subnet snet-web \
  --nsg "" \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-web.yaml
```

The data VM — **no public IP at all**, in `snet-data`:

```bash
az vm create \
  --resource-group rg-network-<yourinitials> \
  --name vm-data \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --vnet-name vnet-catalog \
  --subnet snet-data \
  --nsg "" \
  --public-ip-address "" \
  --admin-username azureuser \
  --generate-ssh-keys
```

> **Note:** If a size or capacity error appears, add `--location westus3` to the group in Step 2 and rebuild, or ask your instructor.

**Why `--nsg ""`?** Left to itself, `az vm create` makes a fresh NSG for each VM's network interface — that is the "quietly created" behavior from the last lab. The empty string says: create no NIC-level NSG; this network is governed at the subnet, where we put the rules on purpose. One layer of rules, in one predictable place.

**Why `--public-ip-address ""` on the data VM — isn't the NSG protection enough?** This is **defense in depth**, and the distinction is worth being precise about. An NSG is a *rule* that filters traffic arriving at an address. No public IP means there is *no address for internet traffic to arrive at* — nothing to probe, nothing to misconfigure, no rule that a bad change next year could accidentally loosen. The strongest control is the absent attack surface. Databases, internal APIs, and anything else with no business being on the internet should simply not have a public address.

While the VMs build (a few minutes), notice each `vm create` returns a `privateIpAddress` from the subnet you placed it in — `10.20.1.x` for web, `10.20.2.x` for data. Save the data VM's private IP; the proofs need it.

---

## Step 6 – Proof #1 and #2: The Internet's View

Get the web VM's public IP:

```bash
az vm show --resource-group rg-network-<yourinitials> --name vm-web \
  --show-details --query publicIps --output tsv
```

**Proof #1 — the web tier serves the internet.** Browse to the IP, or:

```bash
curl http://<web-public-ip>
```

The web tier page appears: `allow-http` at work.

> **If the connection is refused:** cloud-init may still be installing nginx — it runs for a minute or two after `vm create` returns. Wait and retry.

**Proof #2 — the data tier does not exist, as far as the internet is concerned.** Ask Azure for the data VM's public address:

```bash
az vm show --resource-group rg-network-<yourinitials> --name vm-data \
  --show-details --query publicIps --output tsv
```

Empty. There is no address to attack. `10.20.2.x` is a private address — the internet cannot route to it, no matter what any firewall rule says.

> **Checkpoint:** The web page loads from the public IP, and `vm-data` has no public IP to try.

---

## Step 7 – Proof #3: The Web Tier Can Reach the Data Tier

SSH into the web VM:

```bash
ssh azureuser@<web-public-ip>
```

From **inside vm-web**, test whether the data VM answers on its SSH port (substitute the private IP you saved):

```bash
nc -zv 10.20.2.<x> 22
```

> **Note:** If `nc` is not found on the VM, install it first: `sudo apt install -y netcat-openbsd`

Expected: `Connection to 10.20.2.<x> 22 port [tcp/ssh] succeeded!` (the first VM in a subnet typically gets `.4`) — the web tier reaches the data tier. Stay logged in to vm-web.

**Hold on — `nsg-data` is empty. Why was that allowed?** Because an NSG is never truly empty. Every NSG is born with **default rules** at priorities 65000+ — you can see them from Cloud Shell (open a second tab, or run this later):

```bash
az network nsg rule list \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-data \
  --include-default \
  --output table
```

The three inbound defaults, in order:

| Priority | Rule | Effect |
|---|---|---|
| 65000 | `AllowVnetInBound` | Traffic from inside the VNet is allowed |
| 65001 | `AllowAzureLoadBalancerInBound` | Azure's health probes are allowed |
| 65500 | `DenyAllInBound` | **Everything else is denied** |

Your `nc` test matched `AllowVnetInBound` — both VMs are in the same VNet, so they trust each other by default. Meanwhile the final rule, `DenyAllInBound`, is why an empty NSG still blocks the internet: deny-by-default is built into the bottom of every NSG. Your job as a designer is deciding what to permit *above* it — and, sometimes, tightening the VNet-wide trust, which is exactly what you do next.

---

## Step 8 – Proof #4: Break the Path, Then Beat the Deny with Priority

The security lead's standard is stricter than "the VNet trusts itself." Suppose the data tier should refuse SSH even from inside the VNet. From **Cloud Shell** (not the vm-web session — keep that open):

```bash
az network nsg rule create \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-data \
  --name deny-ssh \
  --priority 200 \
  --direction Inbound \
  --access Deny \
  --protocol Tcp \
  --source-address-prefixes 10.20.0.0/16 \
  --destination-port-ranges 22
```

Back in your **vm-web** session, run the same test:

```bash
nc -zv -w 5 10.20.2.<x> 22
```

Expected: it fails (times out after the 5-second wait). Your `deny-ssh` at priority 200 now outranks `AllowVnetInBound` at 65000 — the rule with the **lower number wins**, and evaluation stops at the first match.

Now carve the one exception the design actually calls for — the web subnet, and only the web subnet, may SSH to the data tier:

```bash
az network nsg rule create \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-data \
  --name allow-ssh-from-web \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.20.1.0/24 \
  --destination-port-ranges 22
```

From vm-web, test one final time:

```bash
nc -zv 10.20.2.<x> 22
```

It succeeds again. Read the data tier's rulebook top to bottom and confirm you can predict every outcome:

| Priority | Rule | Says |
|---|---|---|
| 100 | `allow-ssh-from-web` | SSH from `10.20.1.0/24` — allowed |
| 200 | `deny-ssh` | SSH from anywhere else in the VNet — denied |
| 65000 | `AllowVnetInBound` | All other VNet traffic — allowed |
| 65500 | `DenyAllInBound` | Everything else — denied |

**Why is this priority mechanism worth drilling?** Because every real NSG is read exactly this way, and most real network puzzles ("why can't the app reach the database?") are solved by reading the rules in priority order and finding the first match. You have now built, broken, and precisely repaired a path using nothing but priorities — with the change taking effect in seconds, on running VMs, without touching either machine. That last part is the software-defined network earning its name.

Type `exit` to leave vm-web.

> **Checkpoint:** You watched the same test succeed, fail, and succeed again as rules changed — and you can explain each result by pointing at the first matching rule.

---

## Step 9 – Refine a Rule with an Application Security Group

Your `allow-ssh-from-web` rule from Step 8 hard-codes `10.20.1.0/24` — the web subnet's address range — as its source. That works today, but address ranges are exactly the kind of detail that changes: a re-subnet, an added address space, a second web subnet in another zone, and the CIDR in that rule is suddenly wrong or incomplete. Fix that by naming the *role* instead of the *address*.

Create an **Application Security Group** and put the web VM's network interface in it:

```bash
az network asg create --resource-group rg-network-<yourinitials> --name asg-web

az network nic ip-config update \
  --resource-group rg-network-<yourinitials> \
  --nic-name vm-webVMNic \
  --name ipconfigvm-web \
  --application-security-groups asg-web
```

> **Note:** If `vm-webVMNic` is not the exact NIC name, find it first with `az vm show --resource-group rg-network-<yourinitials> --name vm-web --query "networkProfile.networkInterfaces[0].id" --output tsv` and take the last path segment. The IP-configuration name defaults to `ipconfig<vmname>` (so `ipconfigvm-web` here); if yours differs, look it up with `az network nic show --resource-group rg-network-<yourinitials> --name vm-webVMNic --query "ipConfigurations[0].name" --output tsv`.

Now replace the CIDR-based rule with an ASG-based one:

```bash
az network nsg rule delete \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-data \
  --name allow-ssh-from-web

az network nsg rule create \
  --resource-group rg-network-<yourinitials> \
  --nsg-name nsg-data \
  --name allow-ssh-from-web-asg \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs asg-web \
  --destination-port-ranges 22
```

SSH into vm-web again and repeat the same test from Step 8 to confirm nothing broke:

```bash
ssh azureuser@<web-public-ip>
nc -zv 10.20.2.<x> 22
exit
```

**Why should a rule name a role instead of an address?** Because `asg-web` means "any NIC I have tagged as part of the web tier," while `10.20.1.0/24` means "any address in this specific range, tagged or not." The moment you add a second web subnet, spin up a VM outside the original range, or renumber the network during a redesign, the CIDR rule silently stops matching what you meant, while the ASG rule keeps matching exactly the machines you intended — because membership, not address, is what it checks. This is the same shift that tags made for organizing resources, applied to network rules: a name that survives infrastructure changes instead of a number that does not.

**Why is this a small change with a large payoff at scale?** In a two-VM lab, the CIDR and the ASG produce an identical result — you just proved that. In a production network with dozens of subnets and hundreds of NICs, they diverge fast: an audit of "which NSG rules affect the web tier" becomes a search for the string `asg-web` instead of a spreadsheet of every CIDR block anyone has ever assigned to that tier. Reach for ASGs the moment a rule's true subject is "a role," not "a range."

> **Checkpoint:** SSH from vm-web to the data VM still succeeds, now permitted by group membership instead of an address range.

---

## Step 10 – Scale the Web Tier and Put a Load Balancer in Front of It

Right now the web tier is one VM with one public IP — if `vm-web` goes down, the catalog site goes down with it, no matter how well-designed the NSGs around it are. Fix the actual availability problem: add a second web VM and a load balancer in front of both.

First, a small change to the cloud-init file so each VM's page identifies itself — useful for proving the load balancer is really splitting traffic:

```bash
cd ~/network-lab

cat > cloud-init-web2.yaml << 'EOF'
#cloud-config
package_update: true
packages:
  - nginx
runcmd:
  - systemctl enable --now nginx
  - 'echo "<h1>Cascade Outfitters - Web Tier</h1><p>Served by: $(hostname)</p>" > /var/www/html/index.html'
EOF
```

Create the second web VM in the same subnet:

```bash
az vm create \
  --resource-group rg-network-<yourinitials> \
  --name vm-web2 \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --vnet-name vnet-catalog \
  --subnet snet-web \
  --nsg "" \
  --public-ip-address "" \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-web2.yaml
```

**Why no public IP on the second web VM?** Because it no longer needs one. The load balancer is about to become the single public entry point for the whole tier; individual web VMs answering the internet directly would let traffic bypass the load balancer's health checks entirely. This is the horizontal-scaling pattern from the compute lab's vertical-scaling step, completed: instead of one bigger machine, you now have two identical, replaceable machines, fronted by something that decides which one answers each request.

Add `vm-web2` to the `asg-web` group you created in Step 9, so it inherits the same data-tier access as `vm-web`:

```bash
az network nic ip-config update \
  --resource-group rg-network-<yourinitials> \
  --nic-name vm-web2VMNic \
  --name ipconfigvm-web2 \
  --application-security-groups asg-web
```

**Why is this line not optional?** Because "two identical, interchangeable web-tier members" has to be *true*, not just aspirational. The `allow-ssh-from-web-asg` rule on `nsg-data` permits traffic from `asg-web` members only. If `vm-web2` is left out of the group, it silently loses the data-tier access `vm-web` has — the two machines look interchangeable to the load balancer but behave differently the moment either one needs to reach the database. This is exactly the kind of drift ASGs are meant to prevent, and it only works if every new member of the tier joins the group. Membership is now the single thing that defines the web tier, so joining it is part of creating a web VM, not an afterthought.

Now build the load balancer — a public frontend, a backend pool holding both web VMs, a health probe, and a rule connecting them:

```bash
az network public-ip create \
  --resource-group rg-network-<yourinitials> \
  --name pip-lb-catalog \
  --sku Standard \
  --allocation-method Static

az network lb create \
  --resource-group rg-network-<yourinitials> \
  --name lb-catalog \
  --sku Standard \
  --public-ip-address pip-lb-catalog \
  --frontend-ip-name feConfig \
  --backend-pool-name web-pool

az network nic ip-config address-pool add \
  --resource-group rg-network-<yourinitials> \
  --nic-name vm-webVMNic \
  --ip-config-name ipconfigvm-web \
  --lb-name lb-catalog \
  --address-pool web-pool

az network nic ip-config address-pool add \
  --resource-group rg-network-<yourinitials> \
  --nic-name vm-web2VMNic \
  --ip-config-name ipconfigvm-web2 \
  --lb-name lb-catalog \
  --address-pool web-pool

az network lb probe create \
  --resource-group rg-network-<yourinitials> \
  --lb-name lb-catalog \
  --name http-probe \
  --protocol Tcp \
  --port 80

az network lb rule create \
  --resource-group rg-network-<yourinitials> \
  --lb-name lb-catalog \
  --name http-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name feConfig \
  --backend-pool-name web-pool \
  --probe-name http-probe
```

**Read that sequence as a sentence.** A **frontend** is the one public address callers use. A **backend pool** is the set of machines that can answer. A **probe** is how the load balancer knows which pool members are healthy right now. A **rule** ties the three together: traffic arriving at the frontend on port 80 goes to a healthy member of the backend pool, chosen by port 80. Remove any one piece — no probe, no pool member — and the rule has nothing to send traffic to.

**Why does the probe matter as much as the rule?** Because the probe is the difference between a load balancer and a static list of servers. Every few seconds, the load balancer's probe checks port 80 on each pool member; a VM that stops answering is quietly removed from rotation without you doing anything, and added back the moment it recovers. This is the mechanism behind "the site survived one server crashing" — not because the crashed server suddenly became fine, but because the load balancer stopped sending it traffic within seconds of it going unhealthy.

The subnet's NSG (`nsg-web`) already allows inbound port 80 from `Internet` — that rule governs the whole subnet, so both VMs already inherit it without any change. Confirm the two web VMs no longer need their own open door: notice `nsg-web`'s rules were written once, in Step 3, for a tier that then grew from one VM to two without a single rule changing. That is subnet-level policy paying for itself.

Now hit the load balancer's public IP, several times, and watch which VM answers:

```bash
LB_IP=$(az network public-ip show --resource-group rg-network-<yourinitials> --name pip-lb-catalog --query ipAddress --output tsv)
for i in 1 2 3 4 5 6; do curl -s http://$LB_IP; done
```

> **If every response shows the same hostname:** the load balancer distributes by a hash of the connection's source and destination, not strict round-robin, so a handful of requests from the same shell can land on the same backend. Wait a few seconds and run the loop again, or add `-H "Cache-Control: no-cache"`; over enough requests both VMs will appear.

**Why is seeing both hostnames the actual proof, not just a nice touch?** Because it demonstrates the load balancer is genuinely distributing load across independent machines rather than just proxying to one. This is *elasticity's* physical partner from the slides: elasticity says capacity should grow and shrink with demand, and a load balancer with a backend pool is the mechanism that makes adding the next VM — for the next holiday sale — a `nic ip-config address-pool add` command instead of a redesign.

> **Checkpoint:** Repeated curls to the load balancer's IP return the catalog page from both `vm-web` and `vm-web2`, identified by hostname.

---

## Step 11 – Map the Rest of the Networking Stack Onto What You Built

Your two-subnet design, now with a load balancer and an ASG in it, is the skeleton of a production network. Everything else in Azure networking is an upgrade to a specific part of it — and now that you have built the skeleton, each one has an obvious place. Walk the diagram from the internet inward:

| Where in your design | Service | What it would add |
|---|---|---|
| The load balancer you just built | **Azure Load Balancer** *(built)* | You saw it work at the connection level (TCP/UDP), splitting traffic across a pool by health. Scaling the pool from two VMs to twenty needs no change to the rule |
| Same place, smarter | **Application Gateway** | A load balancer that understands HTTP: it can route `/api` and `/images` to different backends and inspect requests for attacks (its Web Application Firewall) |
| In front of *everything*, globally | **Azure Front Door / CDN** | When Cascade Outfitters goes multi-region, Front Door routes each user to the nearest healthy region, and the CDN caches the catalog images close to users worldwide |
| Around the whole VNet | **Azure Firewall** and **DDoS Protection** | Your NSGs filter by port and address per subnet; Azure Firewall is a central, managed chokepoint with organization-wide rules, and DDoS Protection absorbs flood attacks before they reach anything |
| Between this VNet and the office | **VPN Gateway / ExpressRoute** | The catalog team's on-premises systems reach `10.20.x.x` privately — VPN Gateway over encrypted internet, ExpressRoute over a dedicated private circuit for guaranteed bandwidth |
| Between this VNet and another VNet | **VNet peering** | When another team's VNet needs to talk to yours, peering connects the two private address spaces directly — no public internet involved |
| Inside `snet-data` | **Private Endpoints / Private Link** | When the team replaces `vm-data` with a managed database (Day 2), a private endpoint gives that PaaS service a private IP *inside your subnet* — so even a managed service is never internet-facing |
| Naming everything | **Azure DNS** | Private DNS zones let vm-web reach the data tier as `db.catalog.internal` instead of a hard-coded `10.20.2.4` |
| Your `nsg-data` rule | **Application Security Groups** *(built)* | You already replaced a CIDR-based rule with a role-based one — the same pattern scales to every tier as the network grows |

**Why learn placements instead of the services themselves?** Because the hard part of these services is never their configuration — it is knowing *where they belong and what problem they solve*. That is exactly what your two-subnet skeleton, now genuinely load-balanced, gives you. When someone says "we should put Front Door in front of the catalog," you can now locate that sentence on a diagram you built with your own hands, next to a load balancer you already stood up yourself. The deeper dives come later; the mental map is what Day 1 owes you.

**Notice the recurring theme.** Load Balancer versus Application Gateway, NSG versus Azure Firewall, VPN versus ExpressRoute — each pair is the same trade you measured in the compute lab: the simpler tool versus the more capable, more expensive one. Azure rarely gives you one option; it gives you a spectrum and expects you to justify your position on it.

---

## Step 12 – Clean Up

Three running VMs bill by the hour, plus the load balancer's public IP. One group holds everything:

```bash
az group delete --name rg-network-<yourinitials> --yes --no-wait
```

Verify:

```bash
az group list --output table
```

**What is being torn down?** Run `az resource list --resource-group rg-network-<yourinitials> --output table` before the delete finishes and count: three VMs, three disks, three NICs, two public IPs, two NSGs, an ASG, a load balancer, and the VNet — sixteen or so resources from one lab, several of them (the load balancer, the ASG membership) depending on others existing first. This is why the lifecycle habit matters: nobody wants to hand-delete sixteen interdependent resources in the right order. The group does it for you, in one call, regardless of what depends on what.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **VNet** | Your private, isolated address space — `10.20.0.0/16`, planned roomier than today's need |
| **Subnets** | Tiers of the design (`snet-web`, `snet-data`) and the natural attachment point for policy |
| **NSG** | A stateful firewall as a prioritized rule list; lower number wins, first match stops evaluation |
| **Service tags** | Names like `Internet` that stand in for maintained IP ranges |
| **Default rules** | Every NSG ends in `DenyAllInBound` — deny-by-default is built in; `AllowVnetInBound` makes the VNet trust itself until you say otherwise |
| **Subnet vs NIC attachment** | Subnet-level rules govern every current and future VM in the tier |
| **No public IP** | The strongest network control is an absent attack surface — defense in depth beyond any rule |
| **Software-defined networking** | Paths broken and repaired in seconds by rule changes, no cables, no reboots |
| **Application Security Groups** | Rules written against a role (`asg-web`) instead of an address range — survive re-subnetting and scale-out unchanged |
| **Load Balancer** | Frontend, backend pool, health probe, and rule — traffic only reaches members the probe currently sees as healthy |
| **Horizontal scaling in practice** | Two VMs behind a load balancer survived one VM's traffic being redirected without a config change — elasticity from the slides, built by hand |
| **The wider stack** | Application Gateway, Front Door/CDN, Firewall/DDoS, VPN/ExpressRoute, peering, Private Endpoints, and DNS each upgrade one specific spot in the design you built |

---

## Reflection

Answer for yourself or discuss with a partner:

1. The data VM had an *empty* NSG, yet the internet could not reach it — for two independent reasons. What are they, and why is having both better than either alone?
2. A teammate proposes giving `vm-data` a public IP "just temporarily, for debugging." What would you check or insist on before agreeing — if you agree at all?
3. In Step 8, the same packet was allowed, then denied, then allowed again. Nothing changed on either VM. Where does the enforcement actually happen?
4. The web tier will grow to five VMs next month. What, if anything, must change in your NSG rules for them to be governed identically? Why?
5. You rewrote `allow-ssh-from-web` to use `asg-web` instead of `10.20.1.0/24`, and the test behaved identically. What situation, a month from now, would make the two versions of that rule behave *differently*?
6. Your load balancer's probe checks port 80 every few seconds. If `vm-web2`'s nginx crashed but the VM kept running, what would the probe do, and how quickly would customers stop being routed to it?

---

## Optional Challenge

A real catalog database listens on a database port — say TCP **5432** (PostgreSQL) — not port 22. Two exercises:

1. **Predict, then test.** Read the data tier's rulebook from Step 8 and predict: from vm-web, is port 5432 on the data VM currently reachable, blocked, or something else? Then test it: `nc -zv -w 5 <data-ip> 5432`. The answer is *connection refused* — which is neither "reachable" nor "blocked." Your `deny-ssh` rule only covers port 22, so 5432 sails through `AllowVnetInBound` (65000), *arrives* at the VM… and finds no database listening. Learn the distinction: a **timeout** means a firewall dropped the packet; **refused** means the packet arrived and nothing was listening. "Blocked" versus "nothing there" is one of the most useful diagnostics in network troubleshooting.

2. **Now lock the port down properly.** The default VNet-wide allow means 5432 is currently open to *any* future VM in the VNet, not just the web tier. Fix that the same way Step 8 fixed SSH: add a `Deny` rule for 5432 from `10.20.0.0/16` at priority 210, and an `Allow` for 5432 from `10.20.1.0/24` only at priority 110. Retest from vm-web (still *refused* — it arrives, nothing listens), and explain why a test from any future non-web subnet would now *time out* instead.

Clean up when done.

---

## Conclusion

You gave the security lead everything asked for, with packet-level proof: a web tier answering the internet on exactly one chosen port, a data tier with no public address at all, and a private path between tiers governed by rules you wrote, broke, and repaired on purpose. You then went further than the original brief: you replaced a brittle address-based rule with a role-based one that survives redesign, and you turned a single point of failure into a load-balanced, horizontally scaled tier whose NSG rules never needed to change as it grew. Along the way you met the machinery behind all of it — address spaces, subnets as policy boundaries, NSG priority order, default deny, health probes, and backend pools — which is the same machinery behind every production network in Azure, at any scale.

That completes Day 1: you can navigate Azure, govern it, choose a compute model with evidence, and design the network around it. Day 2 builds on this foundation with storage, databases, identity, and the governance and monitoring tools that keep all of it secure and affordable.
