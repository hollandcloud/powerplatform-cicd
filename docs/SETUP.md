# Setup — GitHub Actions, Service Principal & Secrets

This is the one-time setup that gets the three pipelines working. Steps 1–4 are manual
(they create identities and protection rules that can't be scripted from the dashboard);
step 5 can be done **automatically** by the dashboard.

> **Time:** ~20–30 min. You need: Entra ID app-registration rights, Power Platform admin
> on both environments, and admin on the GitHub repo.

---

## 0. Create the repo and activate the workflows

Push this project to a new GitHub repository.

> **Important:** the workflows ship under `examples/workflows/` so they don't run in the
> template repo. **Copy them into `.github/workflows/`** — they must end up at
> `.github/workflows/*.yml` on the **default branch** (`main`), because `workflow_dispatch`
> only lists workflows that exist on the default branch. Keep the filenames unchanged; the
> dashboard triggers each workflow by filename.

```bash
git init
mkdir -p .github/workflows
cp examples/workflows/*.yml .github/workflows/
git add .
git commit -m "Initial deployment dashboard"
git branch -M main
git remote add origin https://github.com/<owner>/<repo>.git
git push -u origin main
```

---

## 1. Register the service principal (Entra ID)

The pipelines authenticate to Dataverse as an app, not a person.

1. **Entra ID → App registrations → New registration.** Name it e.g. `pp-deploy-sp`.
   Single tenant is fine. Register.
2. Copy the **Application (client) ID** → this is `PP_CLIENT_ID`.
3. Copy the **Directory (tenant) ID** → this is `PP_TENANT_ID`.
4. **Certificates & secrets → New client secret.** Copy the **Value** immediately
   (shown once) → this is `PP_CLIENT_SECRET`.

No API permissions/admin consent are required for Dataverse — access is granted by adding
the app as an *application user* inside each environment (next step).

---

## 2. Add the app as an application user in **the source and every destination**

Do this in **each** Dataverse environment you'll export from or deploy to (Dev, Test,
Staging, Prod, …). For destinations in a **different tenant**, you'll use a separate service
principal for that tenant — see [MULTIPLE-ENVIRONMENTS.md](MULTIPLE-ENVIRONMENTS.md).

1. **[Power Platform admin center](https://admin.powerplatform.microsoft.com)** → pick the
   environment → **Settings → Users + permissions → Application users → + New app user.**
2. **Add an app**, search for the registration from step 1, select it.
3. Set the **Business unit** (root) and assign a **security role**:
   - **System Administrator** (simplest), or **System Customizer** plus solution
     import/export privileges.
4. Create.

> If a deploy later fails with `403 / The user is not a member of the organization`, the
> app user is missing or under-privileged in that specific environment — revisit this step.

Note each environment's URL (e.g. `https://contoso-dev.crm.dynamics.com`). The source URL
becomes `DEV_ENV_URL`; your primary destination becomes `PROD_ENV_URL`. Additional
same-tenant destinations need no secret — the dashboard sends their URL with each run.

---

## 3. Create a GitHub Environment per destination (the approval gate)

`deploy-to-prod.yml` and `rollback.yml` deploy **through a GitHub Environment** named by the
`target_environment` input (default `production`). The matching environment makes the job
**pause until a reviewer approves**, and can hold environment-scoped secrets.

1. Repo **Settings → Environments → New environment** → name it to match a destination's
   **GitHub Environment** field in the dashboard (e.g. `production`, `staging`, `test`,
   `client-acme`).
2. Enable **Required reviewers** and add the people allowed to approve deploys to that env.
3. (Optional) restrict deployment branches to `main`.
4. Repeat for each destination you want gated.

> Protected environments need any plan for **public** repos, or **Pro/Team/Enterprise** for
> **private** repos. If a destination's environment doesn't exist, the deploy still runs but
> with **no approval gate** — create it. For destinations in a different tenant, add that
> tenant's service principal as **environment-scoped secrets** here — see
> [MULTIPLE-ENVIRONMENTS.md](MULTIPLE-ENVIRONMENTS.md).

---

## 4. Create a GitHub PAT for the dashboard

The dashboard calls the GitHub REST API to trigger workflows, read run status, and sync
secrets. It needs a token you paste into **Settings** (kept only in browser `localStorage`).

- **Classic PAT:** scopes `repo` + `workflow`.
- **Fine-grained PAT** (scoped to this repo): Contents **R/W**, Actions **R/W**,
  Secrets **R/W**, Deployments **R/W**, Metadata **R**.

---

## 5. Set the five repo secrets

The workflows read these:

| Secret | From | Used for |
|--------|------|----------|
| `PP_CLIENT_ID` | step 1 | service-principal auth |
| `PP_CLIENT_SECRET` | step 1 | service-principal auth |
| `PP_TENANT_ID` | step 1 | service-principal auth |
| `DEV_ENV_URL` | step 2 | default export source (overridable per-run) |
| `PROD_ENV_URL` | step 2 | default deploy target (overridable per-run) |

> **Additional destinations need no new repo secrets** if they're in the same tenant — the
> dashboard sends each destination's URL as the `environment_url` input, and the shared
> service principal authenticates everywhere it's an application user. **Different-tenant**
> destinations use a GitHub Environment with its own scoped secrets instead — see
> [MULTIPLE-ENVIRONMENTS.md](MULTIPLE-ENVIRONMENTS.md).

**Option A — automatic (recommended):** open the dashboard → **Settings** → fill in the
three service-principal fields and the source/destination URLs → **Sync Secrets to GitHub**.
It encrypts each value with the repo's public key (libsodium sealed box) and `PUT`s it via
the API. Click **Check Status** to confirm all five are present. The SP fields are never
stored in the browser — only pushed to GitHub.

**Option B — manual:** repo **Settings → Secrets and variables → Actions → New repository
secret** for each of the five.

---

## 6. Deploy the dashboard web resource

1. In **make.powerapps.com**, open the solution you want it in → **New → More → Web
   resource** (or edit an existing one). Type **HTML (1)**, name it `cx_html_deploydashboard`,
   upload `webresource/cx_html_deploydashboard.html`.
2. **Save → Publish.**
3. Surface it in your app — see [ADD-TO-MODEL-DRIVEN-APP.md](ADD-TO-MODEL-DRIVEN-APP.md).

(CLI alternative: `pac webresource ...` / pack into a solution and import.)

---

## 7. Smoke test

1. Dashboard → **Settings → Check Status** → expect "All 5 secrets configured".
2. Pick a solution → **Export & Validate** → type `EXPORT` → confirm. Watch the run appear
   under *Recent Pipeline Runs*; it should commit ZIPs into `Solutions/`.
3. Pick a destination and version → **Deploy** → confirm. The run shows **awaiting approval**;
   approve it in GitHub; the dashboard flips to running → success and a Release appears.

---

## Workflow inputs (what the dashboard sends)

| Workflow | Inputs |
|----------|--------|
| `export-and-validate.yml` | `solution_name`, `environment_url`, `pac_auth_index` |
| `deploy-to-prod.yml` | `solution_name`, `solution_version`, `target_environment`, `force_overwrite`, `skip_backup`, `environment_url`, `pac_auth_index` |
| `rollback.yml` | `solution_name`, `backup_file`, `target_environment`, `environment_url`, `pac_auth_index` |

`target_environment` is the GitHub Environment (approval gate) for the chosen destination;
`environment_url` is that destination's Dataverse URL. All three workflows can also be run by
hand from the **Actions** tab (`Run workflow`).

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `403 / user is not a member of the organization` | App user missing/under-privileged in that env (step 2) |
| `PP_TENANT_ID secret is not configured` | Secrets not set / not synced (step 5) |
| Deploy runs with no approval prompt | The destination's GitHub Environment doesn't exist (step 3) |
| Deploy hits the wrong tenant / `401` | Different-tenant destination missing env-scoped secrets ([MULTIPLE-ENVIRONMENTS.md](MULTIPLE-ENVIRONMENTS.md)) |
| Dashboard: "GitHub PAT not configured" banner | PAT not entered in Settings (step 4) |
| Workflow not listed for dispatch | Workflow not copied into `.github/workflows/` on `main` (step 0) |
| Rollback "backup not found" | Managed ZIP not present in `Solutions/Backups/` (see README caveats) |
