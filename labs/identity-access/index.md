# Who Can Touch What – Identity, RBAC, and Key Vault

## The Scenario

The catalog now has images, a database, and a network around it — and a problem the previous lab created on purpose. To reach the storage account you pasted a connection string into a shell variable; to reach the database you typed an admin password. Both are secrets, and right now they live in your scrollback and your memory. Meanwhile a new analyst is starting Monday, and your lead has a pointed question:

> "The analyst needs to *read* our cost and resource data — nothing more. Last place I worked, 'give them access' meant handing over the keys to everything, and that is how you end up on the news. I want you to grant exactly what's needed and nothing else, and I want our database password out of people's heads and somewhere access-controlled. Prove the analyst can look but not break."

That is the two-headed discipline of cloud security: **identity and access** (who is allowed to do what, kept to the minimum) and **secrets management** (credentials stored where access can be controlled and audited, not copied around). This lab does both. You will explore your own identity in Microsoft Entra ID, create a scoped **role assignment** that grants read-only access to a resource group, and stand up an **Azure Key Vault** to hold the database connection string from the last lab — retrieving it by permission instead of by memory.

Running through all of it is the Day 1 idea that never leaves: every one of these rules is enforced by Azure Resource Manager at the control plane, so "look but not break" holds no matter which tool the analyst reaches for.

> **Cost note:** Microsoft Entra objects (users, groups, role assignments) are free. Key Vault charges per operation at a fraction of a cent; this lab costs effectively nothing. Everything is deleted in cleanup.

### Prerequisites

- Completion of Day 1, or comfort with Cloud Shell, resource groups, and RBAC basics
- An Azure account with **Owner** on your subscription (required to assign roles — a `Microsoft.Authorization` action). You will create a **service principal** as the analyst identity; this needs no directory-admin role, so it works on a shared class tenant.

---

## Step 1 – See Your Own Identity and What It Can Do

Start with the identity you already have. In Cloud Shell (Bash):

```bash
az ad signed-in-user show --query "{name:displayName, upn:userPrincipalName, id:id}" --output table
```

Now list what that identity is allowed to do at the subscription scope:

```bash
az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) \
  --query "[].{Role:roleDefinitionName, Scope:scope}" --output table
```

You will almost certainly see **Owner** at the subscription scope.

**Why start by looking in the mirror?** Because every access decision in Azure is about a **principal** (a who — a user, a group, or an app), a **role** (a set of allowed actions), and a **scope** (where the role applies). You are a principal; `Owner` is your role; the subscription is your scope. That triple — *who can do what, where* — is the entire mental model of Azure RBAC, and you are about to hand a much smaller version of it to the analyst. Seeing your own assignment first makes the pattern concrete before you create one.

**Why does it matter that you are Owner and not just Contributor?** Because this lab creates a role assignment and a Key Vault, and — as Day 1's lock step foreshadowed — *granting access* is a `Microsoft.Authorization` action that Contributor is denied. Owner (or User Access Administrator) can assign roles; Contributor cannot. The privilege to grant privilege is itself a privilege, held deliberately by few people.

---

## Step 2 – Create the Resource Group the Analyst Will Read

Create the one resource group the analyst identity will be allowed to see:

```bash
az group create \
  --name rg-analyst-<yourinitials> \
  --location centralus \
  --tags project=catalog environment=dev owner=<yourinitials>

RG_ID=$(az group show --name rg-analyst-<yourinitials> --query id -o tsv)
```

Keep the `RG_ID` variable — the next step grants access *scoped to exactly this group* and nothing else.

---

## Step 3 – Create the Analyst Identity with Least-Privilege Access

The analyst needs a **read-only** identity. You will model it with a **service principal** — a non-human identity that a script, tool, or automation signs in as. (Creating a human *user* is a Microsoft Entra ID *directory* operation that requires the **User Administrator** directory role, which subscription Owner does not include. A service principal is an identity you *can* create as Owner, and — unlike a bare user — you can sign in as it yourself to prove least privilege in Step 7.)

Create the service principal and grant it the built-in **Reader** role, scoped to only your resource group, in one command:

```bash
az ad sp create-for-rbac \
  --name "sp-catalog-analyst-<yourinitials>" \
  --role Reader \
  --scopes "$RG_ID"
```

Copy the three values from the output — you need them in Step 7:

| Field | What it is |
|---|---|
| `appId` | the service principal's username |
| `password` | its secret — shown **once** (if lost, reset with `az ad sp credential reset --id <appId>`) |
| `tenant` | your directory (tenant) ID |

**What did that one command do?** It set all three parts of an RBAC grant at once: a **principal** (the service principal), a **role** (`Reader` — view everything, change nothing), at a **scope** (only `rg-analyst-<yourinitials>`). *Who can do what, where* — kept as small as the job allows. That is **least privilege**.

**Why a service principal and not a user?** Because creating directory users needs an Entra role you do not hold as subscription Owner — identity administration and resource administration are separate systems. The service principal is the identity you *can* create, and it is exactly how real automation (a nightly cost report, a monitoring agent) receives scoped access. In production this credential would also be rotated and, where possible, replaced by a **managed identity** so there is no secret to leak.

**A real-world habit:** for *people*, assign the role to a **group** and add each person to it, so access outlives any individual — assign to groups, not to persons. Here the analyst is a single service identity, so you grant it directly.

---

## Step 4 – Read the Grant Back

Confirm the assignment exists and reads the way you intended:

```bash
az role assignment list --scope "$RG_ID" \
  --query "[].{Role:roleDefinitionName, Principal:principalName, Scope:scope}" --output table
```

**Why "Reader," and what does it deliberately withhold?** **Reader** grants view access to everything in its scope and the power to change *nothing* — no create, no update, no delete. It is the purest expression of your lead's "look but not break." Azure ships built-in roles along a spectrum: **Reader** (view), **Contributor** (view + change, but not grant access), **Owner** (everything, including granting access), plus job-specific roles like **Storage Blob Data Reader** or **Cost Management Reader**. Reach for the *least* powerful role that still does the job — the principle of **least privilege**.

**Why scope it to one resource group instead of the subscription?** Because scope is the other half of least privilege. The analyst identity gets Reader on `rg-analyst-<yourinitials>` and *nothing at all* on the rest of the subscription — it cannot even see the other resource groups, let alone the database or storage account. Roles answer "what can they do?"; scope answers "to which resources?" Least privilege needs both dialed down. Because scope follows the Day 1 hierarchy (management group → subscription → resource group → resource), you grant at exactly the altitude a job requires and no higher.

**Why does this bind the identity no matter what tool it uses?** The same reason the Day 1 lock did. This assignment lives in Azure Resource Manager. Portal, CLI, or REST API — every request is checked against the assignment at the control plane. There is no "back door" tool that skips the check — "look but not break" is enforced once, centrally, for all front doors.

> **Checkpoint:** the list shows a **Reader** assignment whose principal is your `sp-catalog-analyst-<yourinitials>`, scoped to your resource group.

---

## Step 5 – Create a Key Vault for the Database Secret

Now the second half of your lead's request: get the database password out of people's heads. Create a **Key Vault**:

Enable the Azure Key Vault provider 
```bash
az provider register --namespace Microsoft.KeyVault
```

```bash
az keyvault create \
  --name kv-catalog-<yourinitials> \
  --resource-group rg-analyst-<yourinitials> \
  --location centralus \
  --enable-rbac-authorization false
```

> **Note:** Key Vault names are **globally unique** (same rule as storage accounts — the vault gets a public endpoint). Add digits if the name is taken.

**What is Key Vault for, and why not just keep secrets in the app's config?** Key Vault is a hardened, access-controlled, audited store for the three kinds of sensitive material an app needs: **secrets** (passwords, connection strings, API keys), **keys** (cryptographic keys for encryption/signing), and **certificates** (TLS certs). The alternative — secrets pasted into config files, environment variables, or shell scripts like the connection string you carried through the last lab — spreads copies everywhere and leaves no record of who read them. A secret in Key Vault lives in exactly one access-controlled place, every access is logged, and the value can be rotated centrally without hunting down copies. It is the professional answer to the risk the last lab deliberately created.

**Why `--enable-rbac-authorization false` — didn't you just praise RBAC?** This exposes a real design choice. Key Vault offers two permission models for its *data* (the secrets themselves): the newer **Azure RBAC** model (grant data roles like *Key Vault Secrets User* through the same RBAC you just used on the resource group) and the older **access policy** model (per-vault lists of who can do what). RBAC is Microsoft's recommended direction, but it has the same control-plane-vs-data-plane wrinkle as storage: being subscription Owner does *not* automatically grant you permission to read the *contents* of a vault — that is a separate data-plane grant. Choosing the access-policy model here (`false`) automatically gives you, the creator, full secret permissions, so the lab runs without a role-propagation wait. The Optional Challenge switches to the RBAC model so you can feel the difference.

---

## Step 6 – Store and Retrieve the Connection String by Permission

Put the database connection string into the vault, then read it back:

```bash
az keyvault secret set \
  --vault-name kv-catalog-<yourinitials> \
  --name CatalogDbConnection \
  --value "Server=tcp:sql-catalog-<yourinitials>.database.windows.net;Database=catalogdb;User ID=catalogadmin;Password=Cascade2026!example;Encrypt=true;"

az keyvault secret show \
  --vault-name kv-catalog-<yourinitials> \
  --name CatalogDbConnection \
  --query value --output tsv
```

(The connection string here is an illustrative value — it mirrors the shape of the one the last lab produced.)

**What changed between the last lab and now?** In the storage lab, the connection string lived in a shell variable — visible to anyone who could see your session, copied wherever the script went, readable with no record. Now it lives in one place, retrieved only by an identity with permission, and every `secret show` is logged. An application would fetch it the same way at startup — by proving its identity to the vault — instead of shipping the secret baked into its code. Nothing about the secret's *value* changed; everything about *who can reach it and whether that is recorded* did. That is the entire point of secrets management.

**Why is "retrieve by permission" the safer pattern even for you, the owner?** Because it collapses the number of copies to one and makes access observable. A password in your head or your scrollback can be shoulder-surfed, committed to git, or pasted into a chat, and you would never know. A secret in Key Vault has an access log, a single source of truth, and a rotation story: change it once in the vault and every authorized reader gets the new value on their next fetch. Owners are not exempt from that benefit — they are the ones who most need the audit trail.

> **Checkpoint:** `secret show` prints the connection string, and you can explain why that is safer than the variable you used in the storage lab even though the value looks identical.

---

## Step 7 – Watch Least Privilege Hold

Prove the grant by *becoming* the analyst identity. Sign in as the service principal, using the `appId`, `password`, and `tenant` you saved in Step 3:

```bash
az login --service-principal \
  --username <appId> \
  --password <password> \
  --tenant <tenant>
```

Now **read** the resource group it was granted — this succeeds:

```bash
az group show --name rg-analyst-<yourinitials> --output table
```

Then try to **change** something in that group — this is refused:

```bash
az storage account create \
  --name stdenied<yourinitials> \
  --resource-group rg-analyst-<yourinitials> \
  --location centralus --sku Standard_LRS
```

Expected: an **`AuthorizationFailed`** error naming the missing write permission. The identity has `Reader`, so it can look but not touch. When done, return to your own identity by clicking the Cloud Shell **restart** (power) icon, or run `az login`.

> **Note:** a brand-new service principal can take a minute or two to become usable for sign-in. In practice Steps 4–6 fill that gap, so by the time you reach this step it should just work; if the login fails immediately, wait a minute and retry.

**Why is this the proof that matters?** Because a permission you have not tested is a permission you are only guessing about. Watching the identity *succeed* at reading and *fail* at writing — at the control plane, whatever the tool — is the difference between "I think least privilege is configured" and "I have seen it hold." Security work lives or dies on that distinction.

> **Checkpoint:** signed in as the service principal, the read succeeds and the write is refused with `AuthorizationFailed`.

---

## Step 8 – Clean Up

First make sure you are signed back in as **yourself**, not the service principal (restart Cloud Shell or `az login` if you have not already). Then remove the resource group and the identity you created:

```bash
az group delete --name rg-analyst-<yourinitials> --yes --no-wait

# The service principal lives in the directory, not the resource group — delete it explicitly:
az ad app delete --id <appId>
```

**Why delete the service principal separately?** Deleting the resource group removes the Key Vault and the Reader assignment scoped to it automatically — but the service principal lives in the **directory**, not in any resource group or subscription. Identity is its own system: resource cleanup does not touch it, so directory objects need their own explicit deletion. Forgetting this is how directories accumulate orphaned identities for years — a governance mess that is also a security one, since every stale credential is a possible way in.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Microsoft Entra ID** | Azure's identity system; answers *authentication* — who you are — separately from the resource hierarchy |
| **Authentication vs authorization** | Entra ID proves who you are; RBAC decides what you may do |
| **Principal / role / scope** | The three-part core of RBAC: *who* can do *what*, *where* |
| **Service principal** | A non-human identity for tools/automation — created here as the analyst, granted scoped Reader, and signed in as to prove least privilege |
| **Groups over people** | For human teams, assign roles to groups so access outlives any individual and stays auditable |
| **Built-in roles** | Reader, Contributor, Owner, and job-specific roles along a least-to-most-power spectrum |
| **Least privilege** | Grant the weakest role at the narrowest scope that still does the job |
| **Key Vault** | An access-controlled, audited home for secrets, keys, and certificates |
| **Control plane vs data plane** | Managing a vault (or storage account) is separate from reading its contents — a distinct grant |
| **Grant-access is privileged** | Only Owner / User Access Administrator can assign roles; Contributor cannot |

---

## Reflection

Answer for yourself or discuss with a partner:

1. You gave the analyst Reader on one resource group, not Contributor on the subscription. Name the two independent dials you turned down, and what each one prevented.
2. You granted the single service identity Reader directly. For a team of *people*, why would you instead assign the role to a **group** and add members to it?
3. The connection string looked identical in a shell variable and in Key Vault. What actually became safer, and for whom?
4. Being subscription Owner did not automatically let you read a Key Vault's secret contents under the RBAC model. Explain the control-plane-vs-data-plane distinction in your own words.
5. Deleting the resource group removed the role assignment but not the service principal. Why, and what cleanup habit does that imply?

---

## Optional Challenge

Two independent challenges, pick either or both:

1. **Feel the RBAC data-plane model.** Create a second Key Vault with `--enable-rbac-authorization true`. Try `az keyvault secret set` on it immediately — it fails, because being Owner does not grant *data* access under the RBAC model. Grant yourself the **Key Vault Secrets Officer** role at the vault's scope (`az role assignment create --role "Key Vault Secrets Officer" --scope <vault-id> --assignee <your-id>`), wait a minute for propagation, and try again. Explain why this friction is, counterintuitively, a security feature.
2. **Build a custom least-privilege story.** Suppose the analyst also needs to read *costs* but still nothing else. Assign the built-in **Cost Management Reader** role to the service principal at the **subscription** scope (`az role assignment create --assignee <appId> --role "Cost Management Reader" --scope /subscriptions/<sub-id>`), while keeping resource **Reader** at only the one resource group. Explain how combining two narrow roles beats one broad one.

(Clean up any extra resources, role assignments, and identity objects afterward.)

---

## Conclusion

You answered your lead on both fronts. The analyst can see one resource group and break nothing, anywhere — least privilege enforced by role *and* scope, at the control plane, for every tool. And the database connection string that was living in a shell variable now lives in Key Vault, retrieved by permission and logged on every access, ready for an application to fetch by proving its identity rather than by carrying the secret around.

Two Day 1 threads converge here: the RBAC and control-plane story that started with resource locks, and the "who is responsible for what" of the shared responsibility model — identity and secrets sit squarely on your side of that line. In the next lab you enforce standards *before* mistakes happen: Azure Policy, which will finally make good on the promise from Day 1 that mature organizations *require* tags rather than merely hoping for them — and you will watch a non-compliant deployment get refused at the same control plane that just told the analyst no.
