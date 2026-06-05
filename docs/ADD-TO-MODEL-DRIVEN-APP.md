# Making the dashboard visible

Prereq: `cx_html_deploydashboard` is uploaded as an HTML web resource and **published**
(see [SETUP.md](SETUP.md) step 6).

---

## Option 1 — A dedicated, single-purpose admin app

Best when you want a clean "Deployment Dashboard" entry separate from the business app:

1. **+ New app → Model-driven**, name it **Deployment Dashboard**.
2. **+ Add page → Web resource →** `cx_html_deploydashboard` → set as default page (as above).
3. **Save → Publish → Share** with your admins/release managers only.

They'll see **Deployment Dashboard** in their Power Apps app list and open it in one click.

---

## Option 2 — Sitemap subarea (XML, for source-controlled apps)

If you manage the app's sitemap as XML, add a SubArea that points at the web resource. Make
it the **first** SubArea in the **first** Group/Area to have it load by default.

```xml
<SubArea Id="sa_deploydashboard"
         Url="$webresource:cx_html_deploydashboard"
         VectorIcon="/WebResources/cx_icon_deploy"
         Client="All,Web"
         AvailableOffline="false"
         PassParams="false">
  <Titles>
    <Title LCID="1033" Title="Deployment Dashboard" />
  </Titles>
</SubArea>
```

Place it as the first `<SubArea>` so it's the app's default page, then import/publish the
solution. (Drop `VectorIcon` or point it at an icon web resource you actually have.)

---

## Option 4 — System dashboard wrapper

Create a Dataverse **dashboard** with a single **web resource** component
(`cx_html_deploydashboard`). Useful if you want it to appear under the app's *Dashboards* area rather than as a standalone page.

---

## Notes

- The HTML already detects its host: embedded in a model-driven app it reads `Xrm.WebApi`
  to list solutions and checks for the **System Administrator** role (non-admins get a
  read-only view); in a plain browser it falls back to scanning the GitHub repo. So it
  behaves correctly however you surface it.
- Whichever option you pick, the app is created/edited at **make.powerapps.com**, which
  satisfies "accessible from make."
- Surfacing the dashboard does **not** grant deploy rights — those are gated by the GitHub
  PAT and each destination's GitHub approval environment, plus the in-app System
  Administrator check.