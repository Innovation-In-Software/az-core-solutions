# Who Can Touch What – Identity, RBAC, and Key Vault

## The Scenario

The catalog now has images, a database, and a network around it — and a problem the previous lab created on purpose. To reach the storage account you pasted a connection string into a shell variable; to reach the database you typed an admin password. Both are secrets, and right now they live in your scrollback and your memory. Meanwhile a new analyst is starting Monday, and your lead has a pointed question:

> "The analyst needs to *read* our cost and resource data — nothing more. Last place I worked, 'give them access' meant handing over the keys to everything, and that is how you end up on the news. I want you to grant exactly what's needed and nothing else, and I want our database password out of people's heads and somewhere access-controlled. Prove the analyst can look but not break."

That is the two-headed discipline of cloud security: **identity and access** (who is allowed to do what, kept to the minimum) and **secrets management** (credentials stored where access can be controlled and audited, not copied around). This lab does both. You will explore your own identity in Microsoft Entra ID, create a scoped **role assignment** that grants read-only access to a resource group, and stand up an **Azure Key Vault** to hold the database connection string from the last lab — retrieving it by permission instead of by memory.

Running through all of it is the Day 1 idea that never leaves: every one of these rules is enforced by Azure Resource Manager at the control plane, so "look but not break" holds no matter which tool the analyst reaches for.

> **Cost note:** Microsoft Entra objects (users, groups, role assignments) are free. Key Vault charges per operation at a fraction of a cent; this lab costs effectively nothing. Everything is deleted in cleanup.

### Prerequisites

- Completion of Day 1, or comfort with Cloud Shell, resource groups, and RBAC basics
- An Azure account. **Some steps create Microsoft Entra ID objects** (a user, a group). This works cleanly if you own your tenant (a personal/free account). On a corporate tenant you may lack permission — each such step notes a fallback so you can still complete the lab.

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

## Step 2 – Create a Group for the Analyst Role

You could assign a role straight to the analyst, but you should not. Create a **group** first:

```bash
az ad group create --display-name "Catalog Analysts" --mail-nickname catalog-analysts
```

> **Corporate-tenant fallback:** If this returns an authorization error, you lack directory permission to create groups. Ask your instructor, or skip ahead and in Step 4 assign the role to *your own* user instead of the group — the RBAC lesson still lands.

**Why assign roles to a group instead of directly to the person?** Because people change and roles endure. When the analyst leaves and a replacement starts, you add and remove them from **Catalog Analysts** — the role assignment never moves. When a second analyst joins, they inherit the exact same access by joining the group. Direct-to-person assignments are how organizations end up, years later, with hundreds of one-off grants nobody can audit. Assigning to groups keeps the answer to "who can read the catalog?" as short as "the members of this group." This is the single most important RBAC habit, and it costs one extra command today.

---

## Step 3 – Create the Analyst User

Create the user who will join the group:

```bash
TENANT_DOMAIN=$(az ad signed-in-user show --query userPrincipalName -o tsv | cut -d'@' -f2)

az ad user create \
  --display-name "Cascade Analyst" \
  --user-principal-name "analyst@$TENANT_DOMAIN" \
  --password 'Analyst2026!ChangeMe'

az ad group member add --group "Catalog Analysts" \
  --member-id $(az ad user show --id "analyst@$TENANT_DOMAIN" --query id -o tsv)
```

> **Corporate-tenant fallback:** If user creation is denied, you lack directory permission to create users. Skip creating the user; in Step 4, assign the role to the **Catalog Analysts** group anyway (an empty group is fine to demonstrate scoped assignment) or to your own user, and read the "test least privilege" step as a thought exercise.

**Why does creating a user belong to identity administration, not resource administration?** Notice this command is `az ad ...`, not `az resource ...` or `az group ...` (resource group). It talks to **Microsoft Entra ID** — Azure's identity service (formerly Azure Active Directory) — which is a separate system from the resource hierarchy you have worked in all course. Entra ID answers *authentication* ("who are you?"); RBAC answers *authorization* ("what may you do?"). Every sign-in to the portal, every `az login`, every app calling a Microsoft API authenticates against Entra ID first, and only then does RBAC decide what the authenticated identity can touch. Keeping those two questions separate — identity here, permissions there — is why the same analyst identity can have read-only rights on one subscription and none on another.

**Why the forced password and what would production add?** The temporary password is a lab shortcut. A real account would require the user to set their own on first sign-in and would sit behind **multi-factor authentication (MFA)** and **Conditional Access** — policies that demand a second factor, or block sign-in from risky locations, before the password even matters. Those features (and passwordless sign-in) are how modern identity resists the stolen-password attack that a password alone cannot. You are not configuring them here, but know that the account you just made would, in production, be only the first layer.

---

## Step 4 – Grant Read-Only Access, Scoped to One Resource Group

Create a resource group for the analyst to have access to, then grant the group the built-in **Reader** role — scoped to *only* that group:

```bash
az group create \
  --name rg-analyst-<yourinitials> \
  --location eastus \
  --tags project=catalog environment=dev owner=<yourinitials>

RG_ID=$(az group show --name rg-analyst-<yourinitials> --query id -o tsv)

az role assignment create \
  --assignee "$(az ad group show --group 'Catalog Analysts' --query id -o tsv)" \
  --role "Reader" \
  --scope "$RG_ID"
```

> **Fallback:** if you skipped the group, replace the `--assignee` value with your own user id (`az ad signed-in-user show --query id -o tsv`).

**Why "Reader," and what does it deliberately withhold?** **Reader** is a built-in role that grants view access to everything in its scope and the power to change *nothing* — no create, no update, no delete. It is the purest expression of your lead's "look but not break." Azure ships dozens of built-in roles along a spectrum: **Reader** (view), **Contributor** (view + change, but not grant access), **Owner** (everything including granting access), plus job-specific roles like **Storage Blob Data Reader** or **Cost Management Reader**. You reach for the *least* powerful role that still lets the person do their job — the principle of **least privilege**.

**Why scope it to one resource group instead of the subscription?** Because scope is the other half of least privilege, and it is where the real damage is usually prevented. The analyst gets Reader on `rg-analyst-<yourinitials>` and *nothing at all* on the rest of the subscription — they cannot even see the other resource groups, let alone the database or storage account. Had you granted Reader at the subscription, they could read every secret-adjacent setting everywhere. Roles answer "what can they do?"; scope answers "to which resources?" Least privilege needs both dialed down. Because scope follows the Day 1 hierarchy (management group → subscription → resource group → resource), you can grant at exactly the altitude a job requires and no higher.

**Why does this rule bind the analyst no matter what tool they use?** The same reason the Day 1 lock did. This assignment lives in Azure Resource Manager. Whether the analyst opens the portal, runs the CLI, or calls the REST API, every request they make is checked against this assignment at the control plane. There is no "back door" tool that skips the check — "look but not break" is enforced once, centrally, for all front doors.

> **Checkpoint:** `az role assignment list --scope "$RG_ID" --query "[].{Role:roleDefinitionName, Principal:principalName}" --output table` shows the Reader assignment for the group (or your user).

---

## Step 5 – Create a Key Vault for the Database Secret

Now the second half of your lead's request: get the database password out of people's heads. Create a **Key Vault**:

```bash
az keyvault create \
  --name kv-catalog-<yourinitials> \
  --resource-group rg-analyst-<yourinitials> \
  --location eastus \
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

## Step 7 – (Optional but recommended) Watch Least Privilege Hold

If you created the analyst user and have a moment, prove the grant. Open a **private/incognito browser window**, go to `https://portal.azure.com`, and sign in as `analyst@<your-tenant-domain>` with the temporary password (you may be prompted to change it).

- Open **Resource groups**. The analyst sees **only** `rg-analyst-<yourinitials>` — not `rg-data`, not the others.
- Open that group and try **Create** or **Delete**. Both are refused: Reader can look, not touch.
- Try to open the Key Vault's secret. Even inside the one group they can read, the secret's *data* is gated separately — they were never granted secret access.

**Why is this the proof that matters?** Because a permission you have not tested is a permission you are only guessing about. Watching the analyst *fail* to create, delete, or read secrets — while succeeding at viewing — is the difference between "I think least privilege is configured" and "I have seen it hold." Security work lives or dies on that distinction. Close the private window when done.

> **Checkpoint:** The analyst can view one resource group and change nothing, anywhere.

---

## Step 8 – Clean Up

Remove the resources, the role assignment, and the identity objects:

```bash
az group delete --name rg-analyst-<yourinitials> --yes --no-wait

# Re-derive the tenant domain in case this is a new Cloud Shell session
TENANT_DOMAIN=$(az ad signed-in-user show --query userPrincipalName -o tsv | cut -d'@' -f2)

az ad group delete --group "Catalog Analysts"
az ad user delete --id "analyst@$TENANT_DOMAIN"
```

> Skip the `az ad` lines for any object you did not create. Deleting the resource group removes the Key Vault and the role assignment scoped to it automatically.

**Why does deleting the resource group also clear the role assignment, but not the group and user?** Because the role assignment was *scoped to* that resource group — it is part of the group's governance and dies with it. The Entra ID user and group, though, live in the **directory**, not in any resource group or even any subscription. That is the same separation from Step 3: identity is its own system. Resource cleanup does not touch it, so identity objects need their own explicit deletion. Forgetting this is how directories accumulate orphaned test accounts for years — a small governance mess that is also a security one, since every stale account is a possible way in.

---

## Concepts Summary

| Concept | What you saw |
|---|---|
| **Microsoft Entra ID** | Azure's identity system; answers *authentication* — who you are — separately from the resource hierarchy |
| **Authentication vs authorization** | Entra ID proves who you are; RBAC decides what you may do |
| **Principal / role / scope** | The three-part core of RBAC: *who* can do *what*, *where* |
| **Groups over people** | Assign roles to groups so access outlives any individual and stays auditable |
| **Built-in roles** | Reader, Contributor, Owner, and job-specific roles along a least-to-most-power spectrum |
| **Least privilege** | Grant the weakest role at the narrowest scope that still does the job |
| **Key Vault** | An access-controlled, audited home for secrets, keys, and certificates |
| **Control plane vs data plane** | Managing a vault (or storage account) is separate from reading its contents — a distinct grant |
| **Grant-access is privileged** | Only Owner / User Access Administrator can assign roles; Contributor cannot |

---

## Reflection

Answer for yourself or discuss with a partner:

1. You gave the analyst Reader on one resource group, not Contributor on the subscription. Name the two independent dials you turned down, and what each one prevented.
2. Why assign the Reader role to a group rather than directly to the analyst, when there is only one analyst today?
3. The connection string looked identical in a shell variable and in Key Vault. What actually became safer, and for whom?
4. Being subscription Owner did not automatically let you read a Key Vault's secret contents under the RBAC model. Explain the control-plane-vs-data-plane distinction in your own words.
5. Deleting the resource group removed the role assignment but not the analyst user. Why, and what cleanup habit does that imply?

---

## Optional Challenge

Two independent challenges, pick either or both:

1. **Feel the RBAC data-plane model.** Create a second Key Vault with `--enable-rbac-authorization true`. Try `az keyvault secret set` on it immediately — it fails, because being Owner does not grant *data* access under the RBAC model. Grant yourself the **Key Vault Secrets Officer** role at the vault's scope (`az role assignment create --role "Key Vault Secrets Officer" --scope <vault-id> --assignee <your-id>`), wait a minute for propagation, and try again. Explain why this friction is, counterintuitively, a security feature.
2. **Build a custom least-privilege story.** Suppose the analyst also needs to read *costs* but still nothing else. Find the built-in **Cost Management Reader** role and add it to the Catalog Analysts group at the subscription scope, while keeping resource Reader at only the one resource group. Explain how combining two narrow roles beats one broad one.

(Clean up any extra resources, role assignments, and identity objects afterward.)

---

## Conclusion

You answered your lead on both fronts. The analyst can see one resource group and break nothing, anywhere — least privilege enforced by role *and* scope, at the control plane, for every tool. And the database connection string that was living in a shell variable now lives in Key Vault, retrieved by permission and logged on every access, ready for an application to fetch by proving its identity rather than by carrying the secret around.

Two Day 1 threads converge here: the RBAC and control-plane story that started with resource locks, and the "who is responsible for what" of the shared responsibility model — identity and secrets sit squarely on your side of that line. In the next lab you enforce standards *before* mistakes happen: Azure Policy, which will finally make good on the promise from Day 1 that mature organizations *require* tags rather than merely hoping for them — and you will watch a non-compliant deployment get refused at the same control plane that just told the analyst no.
