# Give the Catalog Real Data – Storage and Databases

## The Scenario

Across Day 1 you built the shell of the Cascade Outfitters catalog: a resource group to hold it, a compute tier to serve it, and a two-tier network with a "data tier" subnet sitting empty behind the web tier. Empty is the operative word. The catalog has nowhere to keep its product photos and nowhere to keep its product records. Today you fill both gaps, and your lead frames it as two distinct problems:

> "Product *images* and product *data* are not the same kind of thing, and they should not live in the same kind of place. Figure out where each belongs, and — this is the part that always bites us later — decide up front how durable each one needs to be and what it costs to keep. I don't want to discover we're paying archive prices for the homepage banner, or losing orders because we cheaped out on redundancy."

That is the whole discipline of cloud storage in two sentences: **match the storage service to the shape of the data, and match the durability and tier to what the data is worth.** In this lab you will create a storage account for the images, learn why blob storage is the right home for them, move a blob between access tiers and watch the cost trade-off, then stand up a managed Azure SQL Database for the records and query it — the data tier from the networking lab, finally real.

> **Cost note:** The storage account costs a fraction of a cent for this lab. The Azure SQL Database uses the **Basic** tier at roughly a cent per hour (about $5/month if left running — so cleanup matters), or the free tier if your account offers it (noted in the step). Everything is deleted in cleanup. Do not skip cleanup.

### Prerequisites

- Completion of Day 1, or comfort with Cloud Shell, resource groups, and tags
- An Azure account (the free account is enough)

---

## Step 1 – Create the Resource Group

In Cloud Shell (Bash), run (use your initials):

```bash
az group create \
  --name rg-data-<yourinitials> \
  --location centralus \
  --tags project=catalog environment=dev owner=<yourinitials>
```

**Why a fresh resource group again?** Same lifecycle habit from Day 1: the image store and the database exist for the same project and will be torn down together, so they share a group and cleanup stays a single command. The tags are muscle memory now — and by the end of the next lab you will see Azure Policy *require* them.

---

## Step 2 – Create a Storage Account

```bash
az storage account create \
  --name stcatalog<yourinitials> \
  --resource-group rg-data-<yourinitials> \
  --location centralus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot
```

> **Note:** Storage account names must be **globally unique**, lowercase, 3–24 characters, letters and digits only. If `stcatalog<yourinitials>` is taken, add a couple of digits.

**Why must the name be globally unique when a resource group name only had to be unique to you?** Because a storage account gets its own internet endpoint — `https://stcatalog<yourinitials>.blob.core.windows.net`. That hostname has to be unique across all of Azure, the same way a website's domain does. A resource group is just an internal label and never becomes a hostname, so it only needs to be unique within your subscription. Any Azure resource whose name becomes part of a public URL (storage accounts, App Service apps, Key Vaults) inherits this global-uniqueness rule.

**What did `--kind StorageV2` choose?** The general-purpose v2 account — the modern default that supports every storage service (blobs, files, queues, tables) and every access tier. You will almost never pick anything else for new work. Older `kind` values exist mainly for backward compatibility.

**What did `--sku Standard_LRS` choose, and why does it matter more than it looks?** The SKU encodes the **redundancy** — how many copies Azure keeps and how far apart. This is your lead's "how durable does it need to be" decision, made concrete:

| SKU | What it means | Survives | Copies |
|---|---|---|---|
| `Standard_LRS` | Locally redundant | A disk or rack failure | 3, in one datacenter |
| `Standard_ZRS` | Zone redundant | A whole datacenter (zone) failing | 3, across availability zones |
| `Standard_GRS` | Geo redundant | A regional disaster | 6, primary region + paired region |
| `Standard_RAGRS` | Geo redundant, read access | A regional disaster, with read access to the copy | 6, and you can read the secondary |
| `Standard_GZRS` | Zone + geo redundant | Both a zone failure and a regional disaster | The strongest, and the priciest |

**Why did we pick LRS for a lab and why would that be wrong for real orders?** LRS is the cheapest and keeps three copies, which is plenty against the everyday failure — a bad disk. But every copy is in one datacenter, so a flood or fire that takes the building takes your data. That is fine for reproducible product images (you have the originals) and reckless for customer orders (you do not). Notice how the redundancy names map directly onto the resilience vocabulary from Day 1: zones for surviving a datacenter, region pairs for surviving a region. Redundancy is just those ideas applied to bytes at rest.

**What did `--access-tier Hot` choose?** The default read/write tier for data you touch often — you will change it in a moment and watch the trade-off appear.

---

## Step 3 – Create a Blob Container and Upload Product Images

Fetch a connection string so the storage commands can authenticate, then create a container and upload some stand-in "images":

```bash
CONN=$(az storage account show-connection-string \
  --name stcatalog<yourinitials> \
  --resource-group rg-data-<yourinitials> \
  --query connectionString --output tsv)

az storage container create --name product-images --connection-string "$CONN"

cd ~ && echo "tent product photo" > tent.jpg && echo "kayak product photo" > kayak.jpg

az storage blob upload --container-name product-images --name tent.jpg --file tent.jpg --connection-string "$CONN"
az storage blob upload --container-name product-images --name kayak.jpg --file kayak.jpg --connection-string "$CONN"

az storage blob list --container-name product-images --connection-string "$CONN" --output table
```

(The files hold text, not real photos — the storage service treats every blob as an opaque bag of bytes, so a text file stands in perfectly for a JPEG here.)

**Why is blob storage the right home for images, rather than the database?** Because a product photo is a large, unstructured *object* — you store it whole and serve it whole; you never run a query like "find the blue pixels." **Blob** (Binary Large OBject) storage is built for exactly this: cheap, massively scalable, HTTP-addressable objects, each reachable by its own URL. A database, by contrast, is built to *query* structured rows — and stuffing megabyte images into database rows is a classic, expensive mistake. Your lead's instinct ("images and data are not the same kind of thing") is this distinction. Two other storage services round out the account you just made: **Azure Files** (a network file share you can mount like a drive) and **queues/tables** (for messaging and simple key-value data) — same account, different data shapes.

**Why did the connection string spare you a login prompt?** The connection string embeds an account key, so every command that carries it is authenticated as the account itself. That is convenient and *exactly the risk* Key Vault solves in the next lab — a key pasted into a variable is a key that can leak. Hold that thought; Lab 6 is partly the answer to "so where should this connection string actually live?"

> **Checkpoint:** `az storage blob list` shows `tent.jpg` and `kayak.jpg` in the `product-images` container.

---

## Step 4 – Move a Blob to a Cooler Tier and See the Trade-Off

Suppose last season's tent is discontinued — its photo must stay available (someone might view an old order) but is rarely served. Move it to the **Cool** tier:

```bash
az storage blob set-tier --container-name product-images --name tent.jpg --tier Cool --connection-string "$CONN"

az storage blob show --container-name product-images --name tent.jpg --connection-string "$CONN" \
  --query "{name:name, tier:properties.blobTier}" --output table
```

**What are the access tiers, and what exactly are you trading?** Access tiers let you pay less to *store* data in exchange for paying more (and waiting longer) to *read* it:

| Tier | Storage cost | Access cost & latency | Use it for |
|---|---|---|---|
| **Hot** | Highest | Lowest, instant | Data served constantly — the current catalog |
| **Cool** | Lower | Higher; 30-day minimum | Infrequently read data — last season's products |
| **Cold** | Lower still | Higher; 90-day minimum | Rarely read, but still needs to be online |
| **Archive** | Lowest | Highest; hours to *rehydrate* before you can read | Compliance copies, old orders you may never open |

**Why is choosing the wrong tier a real bill, not a rounding error?** Because the costs move in *opposite directions*, so a mismatch is expensive both ways. Put the constantly-served homepage banner in Archive and every page load pays a rehydration penalty and stalls for hours. Put seven years of compliance records in Hot and you pay premium storage rates forever for data nobody reads. This is precisely your lead's "archive prices for the homepage banner" fear. The tier is a bet on *how often you will read the data*, and getting the bet right is one of the highest-leverage cost decisions in Azure storage.

**Why does Cool have a 30-day minimum?** Because the tiers assume the bet is real. If you drop a blob to Cool and then delete or re-read it two days later, you are billed as though it stayed 30 days — the discount is for *committing* to leave it cool. Archive's 90-day minimum plus hours-long rehydration is the same idea, turned up. Tiering saves money only when the access pattern genuinely matches the tier.

> **Checkpoint:** `tent.jpg` reports `tier: Cool`; `kayak.jpg` (untouched) is still `Hot`.

---

## Step 5 – Create a Managed SQL Database for the Product Records

Images are handled. Now the structured data — product IDs, names, prices, stock — which *is* a job for a database. This is the data tier from the networking lab, made real, and this time you will not run or patch a database server at all: Azure runs it for you.

Pick a strong admin password and create a logical SQL server, then a database on it:

```bash
SQLPASS='Cascade2026!<pick-your-own>'

# Register the SQL provider. This can take a few minutes to complete. 

az provider register --namespace Microsoft.Sql

az sql server create \
  --name sql-catalog-<yourinitials> \
  --resource-group rg-data-<yourinitials> \
  --location centralus \
  --admin-user catalogadmin \
  --admin-password "$SQLPASS"

az sql db create \
  --resource-group rg-data-<yourinitials> \
  --server sql-catalog-<yourinitials> \
  --name catalogdb \
  --edition Basic
```

> **Note:** The password must be 8+ characters with three of: uppercase, lowercase, digits, symbols. Pick your own and remember it — you need it in Step 7. If your account offers the **free** Azure SQL Database, you may instead replace the `db create` command with the serverless free-tier form (the free offer requires General Purpose serverless Gen5, so `--edition Basic` will not work with it):
>
> ```bash
> az sql db create \
>   --resource-group rg-data-<yourinitials> \
>   --server sql-catalog-<yourinitials> \
>   --name catalogdb \
>   --edition GeneralPurpose --family Gen5 --capacity 2 \
>   --compute-model Serverless \
>   --use-free-limit --free-limit-exhaustion-behavior AutoPause
> ```

**What is a "logical SQL server," and why is it separate from the database?** The server (`sql-catalog-<yourinitials>`) is not a machine you manage — it is an administrative and networking boundary that can hold many databases, sharing one admin login, one firewall, and one connection endpoint. The database is the thing that actually stores rows. Splitting them lets you put a dev and a test database behind the same server and firewall while billing and scaling each database independently.

**Why is this PaaS, and what did you *not* have to do?** Compare this to the VM in the compute lab. You did not choose an OS, install SQL Server, patch it, configure backups, or open a port on a firewall you manage. Azure SQL Database is a **managed** (PaaS) database: Microsoft runs the engine, patches it, backs it up automatically, and keeps it highly available, while you get exactly one thing to care about — your data. This is the same IaaS-vs-PaaS trade from Day 1, now for databases. If you truly needed OS-level control (an unusual SQL Server feature, say), *SQL Server on an Azure VM* is the IaaS option; *SQL Managed Instance* sits in between. For a three-person team, managed is almost always right.

**Where does Cosmos DB fit?** Azure SQL Database is *relational* — tables, rows, SQL. When an app needs a globally distributed, low-latency, schema-flexible (NoSQL) store — a shopping cart hit from five continents, say — **Azure Cosmos DB** is Azure's multi-model answer, and **Azure Database for PostgreSQL / MySQL** are managed versions of those open-source engines. The catalog's tidy product table is a textbook relational fit, so SQL Database is the right call here.

---

## Step 6 – Open the Firewall to Azure Services

By default the SQL server blocks all connections. Allow connections from inside Azure (which includes Cloud Shell and the portal's query editor):

```bash
az sql server firewall-rule create \
  --resource-group rg-data-<yourinitials> \
  --server sql-catalog-<yourinitials> \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

**Why is `0.0.0.0` to `0.0.0.0` an "allow Azure" rule and not "allow everything"?** This is a special-cased rule in Azure SQL: the start and end both being `0.0.0.0` is the documented signal for "permit connections from Azure-internal services," not a literal address range. It is how the browser-based query editor and Cloud Shell reach the database without you hunting down their ever-changing IPs. It is deliberately *not* the same as opening the database to the public internet — for a real workload you would instead allow only specific client IP ranges, or (better) use a **private endpoint** so the database has no public exposure at all, exactly the pattern the networking lab flagged for the data tier.

---

## Step 7 – Create a Table and Query It

Run the queries in the portal's built-in editor — no client tool to install.

1. In the portal, search for **SQL databases** and open **catalogdb**.
2. In the left menu, open **Query editor (preview)**.
3. Sign in with **SQL server authentication**: login `catalogadmin`, password the one you chose in Step 5.
4. Paste this into the editor and click **Run** to create the table and load three products:

```sql
CREATE TABLE products (
    id       INT PRIMARY KEY,
    name     NVARCHAR(100),
    price    DECIMAL(10,2),
    in_stock INT
);

INSERT INTO products (id, name, price, in_stock) VALUES
    (1, 'Trailhead 2-Person Tent', 249.99, 40),
    (2, 'Rapids Kayak',            589.00, 12),
    (3, 'Summit 45L Backpack',     139.50, 75);
```

5. Now clear the editor, paste just the query, and click **Run** again — this is the part that matters:

```sql
SELECT name, price FROM products WHERE in_stock > 20 ORDER BY price DESC;
```

**Why does this feel completely different from the storage account you just used?** Because it *is* a different shape of data doing a different job. You did not fetch whole objects by name — you asked a *question* ("which in-stock products, priced high to low?") and the database computed the answer across rows. That query is the thing a database does and blob storage cannot. Seeing both in one lab is the point: the image lookup and the product query are different operations, so they live in different services, exactly as your lead insisted.

**Why did the query editor work when nothing is installed?** It runs inside the portal and connects to the database over the `AllowAzureServices` rule you added — the portal is an Azure service. It is the fastest way to run SQL against a managed database, and a nice reminder that "managed" reaches all the way to the tooling: no `sqlcmd`, no drivers, no client setup.

> **Checkpoint:** The `SELECT` returns the **Tent (249.99)** and the **Backpack (139.50)** — the two products with stock over 20, most expensive first. The **Kayak is excluded**: its stock is only 12, so it fails the `in_stock > 20` filter (confirm you understand why, even though it is the most expensive product).

---

## Step 8 – Clean Up

The SQL database bills every hour it exists, so this cleanup matters. One group holds everything:

```bash
az group delete --name rg-data-<yourinitials> --yes --no-wait
```

Verify:

```bash
az group list --output table
```

**Why does one command remove a storage account, a SQL server, a database, and a firewall rule all at once?** Because every one of them was created inside `rg-data-<yourinitials>`, and deleting a resource group deletes its entire contents regardless of type — blobs, databases, network rules, all of it. This is the Day 1 lifecycle-container habit doing real work now that the resources genuinely bill. Delete the group and there is nothing left to meter.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Storage account** | The container for blobs, files, queues, tables; its name is a global, internet-facing hostname |
| **Redundancy (SKU)** | LRS/ZRS/GRS/RA-GRS/GZRS — how many copies and how far apart; the durability-vs-cost dial |
| **Blob storage** | The right home for large unstructured objects like images, each addressable by URL |
| **Access tiers** | Hot/Cool/Cold/Archive trade storage cost against access cost and latency — a bet on read frequency |
| **Azure SQL Database** | A managed (PaaS) relational database — Microsoft runs the engine, you own only the data |
| **Logical SQL server** | An admin/network boundary that can hold many databases |
| **SQL firewall** | Deny-by-default; the `0.0.0.0` rule permits Azure-internal callers like the query editor |
| **Service fit** | Match the service to the data shape: blobs for objects, SQL for relational rows, Cosmos for global NoSQL |

---

## Reflection

Answer for yourself or discuss with a partner:

1. Product images went to blob storage and product records went to a SQL database. Give the one-sentence reason each type belongs where it does.
2. You used `Standard_LRS` for the lab. For real customer orders, which redundancy SKU would you argue for, and which Day 1 concept (zones? region pairs?) does your choice map to?
3. A teammate moves the constantly-viewed homepage banner to the Archive tier "to save money." What actually happens to cost and to the user, and why?
4. Setting up the SQL database, you never chose an OS or applied a patch. On the Day 1 shared-responsibility diagram, where is the line for a managed database versus SQL Server on a VM?

---

## Optional Challenge

Two independent challenges, pick either or both:

1. **Automate tiering with a lifecycle policy.** Instead of moving blobs by hand, create a lifecycle management rule that moves any blob untouched for 30 days to Cool and one untouched for 90 days to Archive. In the portal: open the storage account → **Lifecycle management** → add a rule. Explain why this is better governance than manual tiering across ten thousand product images.
2. **Share one image safely with a SAS token.** Generate a read-only, time-limited **SAS (Shared Access Signature)** URL for `kayak.jpg` (`az storage blob generate-sas ... --permissions r --expiry <date> --full-uri`), open it in a browser, and note that it stops working after expiry. Explain how a SAS is safer than handing someone the account connection string — and connect that to why the next lab moves secrets into Key Vault.

(Clean up afterward if you created new resources.)

---

## Conclusion

You solved your lead's two-part problem the way a cloud engineer is supposed to: not "where do I put the files," but "what *shape* is each piece of data, how durable must it be, and what will it cost to keep." Images went to blob storage with a redundancy SKU and an access tier chosen on purpose; records went to a managed SQL database you queried without ever touching a server. The empty data tier from the networking lab now holds something real.

One loose thread runs straight into the next lab: to talk to the storage account and the database, you passed a connection string and typed an admin password. Those are secrets, and right now they live in shell variables and your memory. In the next lab you will fix that — and in doing so meet Microsoft Entra ID, RBAC, and Azure Key Vault, the identity and secrets backbone that decides *who* can touch everything you have built.
