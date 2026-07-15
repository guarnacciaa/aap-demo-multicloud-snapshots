# Testing log

Tracks testing progress for this demo. Update after each session. For procedural verification steps, see [verification.md](verification.md).

**Status values:** `Not tested` · `Pass` · `Partial` · `Fail` · `Blocked`

## Status summary

### Entry playbooks

| Component | Status | Last tested | Notes |
|---|---|---|---|
| CasC apply (`aap_config.yml`) | Not tested | — | |
| CasC cleanup (`aap_cleanup.yml`) | Not tested | — | |
| Smoke test (`verify.yml`) | Not tested | — | |

### Roles

| Component | Status | Last tested | Notes |
|---|---|---|---|
| _(none — no custom roles)_ | — | — | |

### Execution environments

| Component | Status | Last tested | Notes |
|---|---|---|---|
| EE build and push (`ee-multicloud-snapshots`) | Not tested | — | |

### Inventories

| Component | Status | Last tested | Notes |
|---|---|---|---|
| `Demo-Multicloud` (parent, constructed) | Not tested | — | |
| `Azure-Resources` / `azure_vms` | Not tested | — | |
| `AWS-Resources` / `aws_vms` | Not tested | — | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Provision - Azure VM | Not tested | — | Lab/dev mode only. `azure_auth_mode` was missing from `extra_vars` (see Open issues); fix applied 2026-07-15, not yet re-verified |
| Provision - AWS EC2 | Not tested | — | Lab/dev mode only |
| Update - Multicloud inventory hosts | Not tested | — | Lab/dev mode only |
| Snapshot - Connectivity check (dry run) | Fail | 2026-07-15 | Read-only credential/connectivity check. Failed with Azure "Failed to get credentials" (azure_auth_mode: msi not threaded through extra_vars — see Open issues); fix applied same day, not yet re-verified |
| Snapshot - Preview (dry run) | Not tested | — | Read-only pre-flight check. Same `azure_auth_mode` gap fixed 2026-07-15; not yet re-verified |
| Snapshot - Azure by hostname | Not tested | — | Same `azure_auth_mode` gap fixed 2026-07-15; not yet re-verified |
| Snapshot - AWS by hostname | Not tested | — | |
| Snapshot - Verify | Not tested | — | |
| Snapshot - Cleanup preview (dry run) | Not tested | — | Read-only retention policy preview |
| Snapshot - Cleanup (optional) | Not tested | — | |
| Snapshot - Inventory report | Not tested | — | Standalone audit tool, not in the workflow |
| Deprovision - Azure VM | Not tested | — | Lab/dev mode only. Same `azure_auth_mode` gap fixed 2026-07-15; not yet re-verified |
| Deprovision - AWS EC2 | Not tested | — | Lab/dev mode only |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Not tested | — | Lab/dev mode only |
| WF - Multicloud snapshot and retention | Not tested | — | Now starts with connectivity check -> preview, and includes a cleanup preview node before cleanup |
| WF - Demo teardown (destroy infrastructure) | Not tested | — | Lab/dev mode only |

## Open issues

- `azure_auth_mode` was never included in the `extra_vars` of the job templates whose playbooks
  read it (`Snapshot - Connectivity check (dry run)`, `Snapshot - Preview (dry run)`,
  `Snapshot - Azure by hostname`, `Provision - Azure VM`, `Deprovision - Azure VM`).
  `group_vars/all/demo_variables.yml` is not auto-loaded for `playbooks/demo/*.yml` under AAP
  (it runs against AAP's generated inventory, not this repo's `inventory.yml`), so every
  `auth_source` Jinja expression silently fell back to the `service_principal` branch even with
  `azure_auth_mode: msi` set correctly — causing `Snapshot - Connectivity check (dry run)` to fail
  with "Failed to get credentials..." Fixed 2026-07-15 by adding `azure_auth_mode` to the affected
  job templates' `extra_vars`; also added `azure_subscription_id != CHANGE_ME` assertions to
  `aap_config.yml`, `verify.yml`, and `precheck_connectivity.yml` as a related hardening. Re-run
  `aap_config.yml` and re-launch the affected job templates to confirm.
