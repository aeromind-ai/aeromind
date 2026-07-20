# Azure domain cutover — aeromind-ai.com

## Current state (new account)

| Item | Value |
|------|--------|
| Subscription | Azure subscription 1 (`53291c5f-…`) under tenant **AeroMind AI** |
| Resource group | `aeromind-website-rg` |
| Static Web App | `aeromind-website` |
| Default hostname | `wonderful-meadow-0bca93c03.7.azurestaticapps.net` |
| Deploy source | GitHub `aeromind-ai/aeromind` → Actions → `AZURE_STATIC_WEB_APPS_API_TOKEN` |
| Custom domains (pending DNS) | `www.aeromind-ai.com`, `aeromind-ai.com` |

## Old hosting (other Azure account)

| Item | Value |
|------|--------|
| Old SWA hostname | `proud-plant-053b8a11e.1.azurestaticapps.net` |
| Bound name | `www.aeromind-ai.com` |
| DNS provider | **GoDaddy** (`ns47/ns48.domaincontrol.com`) |
| Live check | `https://www.aeromind-ai.com` still serves the **old** site |

That Static Web App is **not** in the current CLI login (only one subscription is visible). It must be cleaned up in the **old Azure account** in the portal.

---

## Step 1 — Remove domain from the old Azure account

1. Sign in to the Azure portal with the account that owns the old site  
   (the one that created `proud-plant-053b8a11e…`).
2. Open **Static Web Apps** → find the app whose default URL is  
   `proud-plant-053b8a11e.1.azurestaticapps.net`  
   (or search custom domain `www.aeromind-ai.com`).
3. **Custom domains** → delete `www.aeromind-ai.com` (and apex if listed).
4. Optional but recommended: **Delete** that Static Web App so it cannot reclaim the domain.
5. Sign out of that account when done.

You do **not** need the old site content — the new site is already deployed.

---

## Step 2 — Add TXT records in GoDaddy (domain ownership)

Azure registered both hostnames with **dns-txt-token** validation. Add these **TXT** records in GoDaddy → DNS management for `aeromind-ai.com`:

| Type | Name / Host | Value | TTL |
|------|-------------|--------|-----|
| TXT | `asuid` | `_che5grpro218t3cfkucyab93ex9jvfp` | 600 |
| TXT | `asuid.www` | `_oxe8pjmaimkiwkv8kwntk4yd8xntew8` | 600 |

Notes:
- On GoDaddy, host for apex is often `@` for other record types, but for Azure SWA the host must be **`asuid`** (apex) and **`asuid.www`** (www).
- If validation tokens expire or you re-create the custom domain, refresh tokens with:

```bash
az staticwebapp hostname show -n aeromind-website -g aeromind-website-rg \
  --hostname aeromind-ai.com --query validationToken -o tsv
az staticwebapp hostname show -n aeromind-website -g aeromind-website-rg \
  --hostname www.aeromind-ai.com --query validationToken -o tsv
```

Wait until status is **Ready** (can take a few minutes after DNS propagates):

```bash
az staticwebapp hostname list -n aeromind-website -g aeromind-website-rg -o table
```

---

## Step 3 — Point traffic at the new Static Web App

### Recommended (matches typical GoDaddy setup)

| Type | Name | Value | TTL |
|------|------|--------|-----|
| CNAME | `www` | `wonderful-meadow-0bca93c03.7.azurestaticapps.net` | 600 |

**Apex (`@`):** keep or set **Domain Forwarding** in GoDaddy:

- Forward `https://aeromind-ai.com` → `https://www.aeromind-ai.com`  
- Prefer **forward only** / permanent (301), with SSL if offered.

That is what the domain effectively does today (apex → GoDaddy forward IPs; real content on `www`).

### After changing the CNAME

Remove any old CNAME/A records that still point `www` at:

`proud-plant-053b8a11e.1.azurestaticapps.net`

---

## Step 4 — Verify

```bash
# DNS
dig +short www.aeromind-ai.com CNAME
# expect: wonderful-meadow-0bca93c03.7.azurestaticapps.net.

# Content (new site headline)
curl -s https://www.aeromind-ai.com/ | grep -o 'before fleet history exists'

# Azure domain status
az staticwebapp hostname list -n aeromind-website -g aeromind-website-rg -o table
```

Also open:

- https://wonderful-meadow-0bca93c03.7.azurestaticapps.net/ (works now)
- https://www.aeromind-ai.com/ (after DNS)
- https://aeromind-ai.com/ (after apex forward)

---

## Continuous deploy

Push to `main` on https://github.com/aeromind-ai/aeromind triggers:

`.github/workflows/azure-static-web-apps.yml` → Azure SWA production.

Secret: `AZURE_STATIC_WEB_APPS_API_TOKEN` (deployment token for `aeromind-website`).

---

## Rollback

1. Point `www` CNAME back to `proud-plant-053b8a11e.1.azurestaticapps.net` (only if that app still exists).
2. Or restore previous GoDaddy DNS snapshot.
