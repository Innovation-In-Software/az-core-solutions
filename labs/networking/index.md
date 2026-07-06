# Build a Segmented Network – VNets, Subnets, and NSGs

## The Scenario

The compute bake-off convinced **Cascade Outfitters** — but it also raised a flag in the architecture review. In the last lab, the CLI quietly created a network around your VM with sensible defaults. The security lead was not thrilled by the word *quietly*:

> "Before this goes anywhere near production, I want the network designed on purpose. Web servers reachable from the internet on exactly the ports we choose. The database tier reachable *only* from the web tier — no public address at all. And I don't want a diagram that claims it works; I want you to prove it with packets."

This lab builds that design from scratch. You will create a virtual network with two subnets — a **web tier** and a **data tier** — place a VM in each, and control every path with **network security groups (NSGs)**. Then you will run the proofs: show the web server answering the internet, show the data server unreachable from the internet, show the web tier reaching the data tier — and then break and restore that path by changing one rule, watching NSG priority decide the winner.

This is the pattern behind almost every real deployment in Azure. Learn it once with two VMs and you will recognize it in systems with two hundred.

> **Cost note:** This lab runs two small VMs, a few cents per hour. Everything is deleted in the cleanup step. Do not skip cleanup.

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
  --location eastus \
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
  --size Standard_B1s \
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
  --size Standard_B1s \
  --vnet-name vnet-catalog \
  --subnet snet-data \
  --nsg "" \
  --public-ip-address "" \
  --admin-username azureuser \
  --generate-ssh-keys
```

> **Note:** If a size or capacity error appears, add `--location westus2` to the group in Step 2 and rebuild, or ask your instructor.

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

## Step 9 – Map the Rest of the Networking Stack Onto What You Built

Your two-subnet design is the skeleton of a production network. Everything else in Azure networking is an upgrade to a specific part of it — and now that you have built the skeleton, each one has an obvious place. Walk the diagram from the internet inward:

| Where in your design | Service | What it would add |
|---|---|---|
| In front of the web tier's single IP | **Azure Load Balancer** | When `vm-web` becomes five VMs, the load balancer gives them one shared address and spreads traffic across the healthy ones. It works at the connection level (TCP/UDP) |
| Same place, smarter | **Application Gateway** | A load balancer that understands HTTP: it can route `/api` and `/images` to different backends and inspect requests for attacks (its Web Application Firewall) |
| In front of *everything*, globally | **Azure Front Door / CDN** | When Cascade Outfitters goes multi-region, Front Door routes each user to the nearest healthy region, and the CDN caches the catalog images close to users worldwide |
| Around the whole VNet | **Azure Firewall** and **DDoS Protection** | Your NSGs filter by port and address per subnet; Azure Firewall is a central, managed chokepoint with organization-wide rules, and DDoS Protection absorbs flood attacks before they reach anything |
| Between this VNet and the office | **VPN Gateway / ExpressRoute** | The catalog team's on-premises systems reach `10.20.x.x` privately — VPN Gateway over encrypted internet, ExpressRoute over a dedicated private circuit for guaranteed bandwidth |
| Between this VNet and another VNet | **VNet peering** | When another team's VNet needs to talk to yours, peering connects the two private address spaces directly — no public internet involved |
| Inside `snet-data` | **Private Endpoints / Private Link** | When the team replaces `vm-data` with a managed database (Day 2), a private endpoint gives that PaaS service a private IP *inside your subnet* — so even a managed service is never internet-facing |
| Naming everything | **Azure DNS** | Private DNS zones let vm-web reach the data tier as `db.catalog.internal` instead of a hard-coded `10.20.2.4` |
| Refining your NSG rules | **Application Security Groups** | Instead of writing rules against IP ranges like `10.20.1.0/24`, ASGs let you tag NICs with names like `asg-web` and write rules against the *names* — the rule survives IP changes and re-subnetting |

**Why learn placements instead of the services themselves?** Because the hard part of these services is never their configuration — it is knowing *where they belong and what problem they solve*. That is exactly what your two-subnet skeleton gives you. When someone says "we should put Front Door in front of the catalog," you can now locate that sentence on a diagram you built with your own hands. The deeper dives come later; the mental map is what Day 1 owes you.

**Notice the recurring theme.** Load Balancer versus Application Gateway, NSG versus Azure Firewall, VPN versus ExpressRoute — each pair is the same trade you measured in the compute lab: the simpler tool versus the more capable, more expensive one. Azure rarely gives you one option; it gives you a spectrum and expects you to justify your position on it.

---

## Step 10 – Clean Up

Two running VMs bill by the hour. One group holds everything:

```bash
az group delete --name rg-network-<yourinitials> --yes --no-wait
```

Verify:

```bash
az group list --output table
```

**What is being torn down?** Run `az resource list --resource-group rg-network-<yourinitials> --output table` before the delete finishes and count: two VMs, two disks, two NICs, a public IP, two NSGs, and the VNet — ten or so resources from one lab. This is why the lifecycle habit matters: nobody wants to hand-delete ten resources in the right dependency order. The group does it for you.

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
| **The wider stack** | Load Balancer, Application Gateway, Front Door/CDN, Firewall/DDoS, VPN/ExpressRoute, peering, Private Endpoints, DNS, and ASGs each upgrade one specific spot in the design you built |

---

## Reflection

Answer for yourself or discuss with a partner:

1. The data VM had an *empty* NSG, yet the internet could not reach it — for two independent reasons. What are they, and why is having both better than either alone?
2. A teammate proposes giving `vm-data` a public IP "just temporarily, for debugging." What would you check or insist on before agreeing — if you agree at all?
3. In Step 8, the same packet was allowed, then denied, then allowed again. Nothing changed on either VM. Where does the enforcement actually happen?
4. The web tier will grow to five VMs next month. What, if anything, must change in your NSG rules for them to be governed identically? Why?

---

## Optional Challenge

A real catalog database listens on a database port — say TCP **5432** (PostgreSQL) — not port 22. Two exercises:

1. **Predict, then test.** Read the data tier's rulebook from Step 8 and predict: from vm-web, is port 5432 on the data VM currently reachable, blocked, or something else? Then test it: `nc -zv -w 5 <data-ip> 5432`. The answer is *connection refused* — which is neither "reachable" nor "blocked." Your `deny-ssh` rule only covers port 22, so 5432 sails through `AllowVnetInBound` (65000), *arrives* at the VM… and finds no database listening. Learn the distinction: a **timeout** means a firewall dropped the packet; **refused** means the packet arrived and nothing was listening. "Blocked" versus "nothing there" is one of the most useful diagnostics in network troubleshooting.

2. **Now lock the port down properly.** The default VNet-wide allow means 5432 is currently open to *any* future VM in the VNet, not just the web tier. Fix that the same way Step 8 fixed SSH: add a `Deny` rule for 5432 from `10.20.0.0/16` at priority 210, and an `Allow` for 5432 from `10.20.1.0/24` only at priority 110. Retest from vm-web (still *refused* — it arrives, nothing listens), and explain why a test from any future non-web subnet would now *time out* instead.

Clean up when done.

---

## Conclusion

You gave the security lead everything asked for, with packet-level proof: a web tier answering the internet on exactly one chosen port, a data tier with no public address at all, and a private path between tiers governed by rules you wrote, broke, and repaired on purpose. Along the way you met the machinery behind it — address spaces, subnets as policy boundaries, NSG priority order, default deny — which is the same machinery behind every production network in Azure, at any scale.

That completes Day 1: you can navigate Azure, govern it, choose a compute model with evidence, and design the network around it. Day 2 builds on this foundation with storage, databases, identity, and the governance and monitoring tools that keep all of it secure and affordable.
