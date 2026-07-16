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
| Snapshot - Connectivity check (dry run) | Pass | 2026-07-15 | Read-only credential/connectivity check. Confirmed against a real AAP instance: Azure MSI auth and AWS IAM auth both succeeded (subscription crif-patchinghq-Int, AWS account 671257380337) |
| Snapshot - Preview (dry run) | Pass | 2026-07-15 | Read-only pre-flight check. Confirmed: Azure VM `secopstest01uat` (1 disk) and AWS instance `crif_rhel10_POC00-RAAP` (1 volume) both previewed successfully |
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

- `Provision - Azure VM`, `Snapshot - Azure by hostname`, and `Deprovision - Azure VM` still need
  a live re-verification of the `azure_auth_mode` extra_vars fix confirmed via
  `Snapshot - Connectivity check (dry run)` and `Snapshot - Preview (dry run)` on 2026-07-15
  (resolved, see below) — they share the affected code path but have not each been individually
  re-launched yet.

## Resolved (2026-07-15)

- **`azure_auth_mode` not threaded through `extra_vars`.** `group_vars/all/demo_variables.yml` is
  not auto-loaded for `playbooks/demo/*.yml` under AAP (it runs against AAP's generated inventory,
  not this repo's `inventory.yml`), so every `auth_source` Jinja expression silently fell back to
  the `service_principal` branch even with `azure_auth_mode: msi` set correctly, causing
  `Snapshot - Connectivity check (dry run)` to fail with "Failed to get credentials...". Fixed by
  adding `azure_auth_mode` to the `extra_vars` of `Snapshot - Connectivity check (dry run)`,
  `Snapshot - Preview (dry run)`, `Snapshot - Azure by hostname`, `Provision - Azure VM`, and
  `Deprovision - Azure VM`. Also added `azure_subscription_id != CHANGE_ME` assertions to
  `aap_config.yml`, `verify.yml`, and `precheck_connectivity.yml` as a related hardening.
  **Confirmed fixed** via a live `Snapshot - Connectivity check (dry run)` run.
- **`azure_rm_subscription_info` id-filter dict/list mismatch.** azure.azcollection 3.18.0's
  get-by-`id` code path returns a single dict from `to_dict()` instead of the one-item list its
  own `RETURN` docs promise, so `precheck_connectivity.yml`'s `subscriptions | length == 1`
  assertion always failed regardless of whether the credentials worked. Fixed by listing all
  accessible subscriptions (`list_items()`, which does return a real list) and checking
  `azure_subscription_id` is a member, instead of filtering by `id:`; also fixed the report
  task's `subscriptions[0]` lookup for the same reason. **Confirmed fixed** via the same live run.
