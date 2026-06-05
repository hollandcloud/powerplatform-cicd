# Solutions/

These folders are **populated by the pipelines at runtime** — they ship empty on purpose
(this repo is the CI/CD *tooling*, not a store of any one environment's solution exports).

| Folder | Written/read by | Contents after first run |
|--------|-----------------|--------------------------|
| `Managed/` | **export-and-validate** writes · **deploy-to-prod** reads | `{solution}_{version}_managed.zip` |
| `Unmanaged/` | **export-and-validate** writes | `{solution}_{version}.zip` |
| `Source/` | **export-and-validate** writes (unpacked) | per-solution unpacked XML for diffing |
| `Backups/` | **rollback** reads | managed ZIPs you stage to restore from |

Run **Export & Validate** from the dashboard once and `Managed/`, `Unmanaged/`, and `Source/`
fill in (and auto-commit). **Deploy** and **Rollback** need a managed ZIP present first — so
Deploy works after one Export; for Rollback, copy the target managed ZIP into `Backups/`.

The `.gitkeep` files just keep the empty directories tracked in git; leave them.
