# Multiple deploy destinations

The dashboard deploys to **as many destination environments as you want** — a classic
promotion chain (Dev → Test → Staging → Prod) or a separate environment per client tenant.
This guide explains the model and how to add destinations.

## The model

- **One source.** Export & Validate always pulls *from* a single source environment (your
  Dev). Configured under **Settings → Export Source**.
- **Many destinations.** Deploy and Rollback target one of a list of destinations, each with
  its own approval gate. Configured under **Settings → Deploy Destinations**.

Each destination has four fields:

| Field | What it is |
|-------|------------|
| **Display Name** | Friendly label shown in the dashboard (e.g. `Staging`, `Client: Acme`). |
| **Environment URL** | The Dataverse org URL the solution is imported into. |
| **GitHub Environment** | The GitHub Environment used as the **approval gate** and (optionally) the holder of environment-scoped secrets. Sent to the workflow as `target_environment`. |
| **PAC Auth Index** | Optional `pac auth select --index` value, rarely needed. |

When you trigger a deploy, the dashboard sends `target_environment` (the GitHub Environment)
and `environment_url` (the Dataverse URL) to `deploy-to-prod.yml`. The job runs
`environment: <target_environment>`, so GitHub applies that environment's reviewers and
scoped secrets.

```
Settings ─► Deploy Destinations
   ┌──────────────────────────────────────────────────────────┐
   │ Test       https://contoso-test.crm.dynamics.com    [test]│
   │ Staging    https://contoso-stg.crm.dynamics.com  [staging]│
   │ Production https://contoso-prod.crm.dynamics.com  [prod...]│
   │ Client Acme https://acme-prod.crm.dynamics.com [client-acme]│
   └──────────────────────────────────────────────────────────┘
```

---

## Case A — same tenant (Dev → Test → Staging → Prod)

The common case: all environments live in the same Microsoft tenant, so **one service
principal** is an application user in all of them. **No new secrets are required.**

1. **Add the app as an application user** in the new environment with a deploy-capable role
   (see [SETUP.md step 2](SETUP.md)).
2. **Dashboard → Settings → Deploy Destinations → + Add destination.** Fill in the display
   name, the environment URL, and a **GitHub Environment** name (e.g. `staging`).
3. **Create the matching GitHub Environment** for the approval gate: repo **Settings →
   Environments → New environment**, name it exactly the value from step 2, add **Required
   reviewers** (see [SETUP.md step 3](SETUP.md)). Skip this only if you want that destination
   ungated.
4. Done. Pick the destination from the **Destination** dropdown on the Deploy/Rollback card.

Why no secrets? The workflow resolves the target URL from the `environment_url` input the
dashboard sends, and authenticates with the repo-level `PP_CLIENT_ID` / `PP_CLIENT_SECRET` /
`PP_TENANT_ID` — the same service principal, which is already an app user in the new env.

---

## Case B — different tenant (e.g. a per-client production)

When a destination is in a **different tenant**, the shared service principal can't reach it.
Use a **GitHub Environment with environment-scoped secrets** — these override the repo-level
secrets for runs that go through that environment.

1. In the **client tenant**, register a service principal and add it as an application user
   in their environment (same as [SETUP.md steps 1–2](SETUP.md)).
2. **Create a GitHub Environment** for the destination (e.g. `client-acme`) and add
   **Required reviewers**.
3. On that environment, add **environment secrets** (repo **Settings → Environments →
   _your env_ → Add secret**):

   | Environment secret | Value |
   |--------------------|-------|
   | `PP_CLIENT_ID` | client tenant's app (client) ID |
   | `PP_CLIENT_SECRET` | client tenant's client secret |
   | `PP_TENANT_ID` | client tenant's directory (tenant) ID |
   | `TARGET_ENV_URL` | client environment URL (optional — the dashboard also sends it) |

4. **Dashboard → Settings → Deploy Destinations → + Add destination** with the client's URL
   and `client-acme` as the **GitHub Environment**.

At deploy time, GitHub injects the environment-scoped `PP_*` secrets in place of the repo
ones, so the workflow authenticates to the client's tenant.

> **Secret precedence:** GitHub environment secrets override repository secrets of the same
> name for jobs that run in that environment — so you don't need different secret *names* per
> tenant, just scope them to the environment.

---

## How the workflow resolves the target

In `deploy-to-prod.yml` and `rollback.yml`:

```yaml
jobs:
  deploy:
    environment:
      name: ${{ inputs.target_environment || 'production' }}   # approval gate + scoped secrets
    # ...
    env:
      PROD_URL: ${{ inputs.environment_url || secrets.TARGET_ENV_URL || secrets.PROD_ENV_URL }}
```

Resolution order for the Dataverse URL:

1. `environment_url` input — what the dashboard sends for the picked destination.
2. `TARGET_ENV_URL` — an (optional) environment-scoped secret.
3. `PROD_ENV_URL` — the repo-level default.

Concurrency is keyed per destination (`group: deploy-<target_environment>`), so a deploy to
Test never blocks a deploy to Prod, while two operations on the **same** destination still
queue safely.

---

## Running it by hand (no dashboard)

Everything the dashboard does is a `workflow_dispatch`. From the **Actions** tab → **Deploy
Solution** → **Run workflow**, set:

- `solution_name`, `solution_version`
- `target_environment` → the GitHub Environment (e.g. `staging`)
- `environment_url` → the destination's Dataverse URL
- `force_overwrite` / `skip_backup` as needed

---

## Gotchas

- **A `target_environment` that has no matching GitHub Environment still runs — with no
  approval gate.** Create the environment to require reviewers.
- **Display Name vs GitHub Environment are different fields.** The display name is cosmetic;
  the GitHub Environment must match a real environment name to gate the run.
- **At least one destination is required** — the dashboard won't let you remove the last one.
- **Rollback reads from `Solutions/Backups/`.** Make sure the managed ZIP you want to restore
  is there before rolling back a destination (see the README caveats).
</content>
