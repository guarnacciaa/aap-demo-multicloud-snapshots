# Testing log

Tracks testing progress for this demo. Update after each session. For procedural verification steps, see [verification.md](verification.md).

**Status values:** `Not tested` · `Pass` · `Partial` · `Fail` · `Blocked`

## Status summary

### Entry playbooks

| Component | Status | Last tested | Notes |
|---|---|---|---|
| CasC apply (`aap_config.yml`) | Not tested | — | Project sync fix (originally 2026-07-15, dropped by commit 7181b71) reinstated 2026-07-21 as `update_project: true` on the project in `projects.yml` instead of the original pre_task — not yet re-verified |
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
| `Demo-MulticloudSnapshots` (parent, constructed) | Not tested | — | |
| `Azure-Resources-MulticloudSnapshots` / `azure_vms` | Not tested | — | |
| `AWS-Resources-MulticloudSnapshots` / `aws_vms` | Not tested | — | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Provision - Azure VM | Not tested | — | Lab/dev mode only. `azure_auth_mode` was missing from `extra_vars` (see Open issues); fix applied 2026-07-15, not yet re-verified |
| Provision - AWS EC2 | Not tested | — | Lab/dev mode only |
| Update - Multicloud inventory hosts | Not tested | — | Lab/dev mode only |
| Snapshot - Connectivity check (dry run) | Pass | 2026-07-15 | Read-only credential/connectivity check. Confirmed against a real AAP instance: Azure MSI auth and AWS IAM auth both succeeded |
| Snapshot - Preview (dry run) | Pass | 2026-07-15 | Read-only pre-flight check. Confirmed: a test Azure VM (1 disk) and a test AWS instance (1 volume) both previewed successfully |
| Snapshot - Azure by hostname | Fail | 2026-07-20 | Failed with `InvalidParameter: The value of parameter snapshot.name is invalid` on the test VM; root cause found and fixed 2026-07-20 (see Resolved), not yet re-verified |
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
- `Snapshot - Azure by hostname` needs a live re-verification of the snapshot-name-length fix
  (resolved 2026-07-20, see below) against the test VM.

## Resolved (2026-07-21)

- **Project sync regression, fixed natively.** The `ansible.controller.project_update` pre_task
  added 2026-07-15 to force-sync the demo project before job template creation (see Resolved
  2026-07-15 below for why it is needed) was silently deleted by commit 7181b71 during an
  unrelated pre_tasks edit — `ansible.controller` stayed in the play's `collections:` list
  (still used by two `post_tasks`), which hid the loss. Discovered when the same class of
  error (`Playbook not found for project.`) recurred on `aap-demo-cloud-native-automation`,
  which never had the fix at all. Rather than re-adding the hand-rolled pre_task, replaced it
  with a cleaner native fix in both demos: `ansible.controller.project`'s own `update_project:
  true` parameter ("force project to update after changes", used with `wait`) set directly on
  the project definition in `group_vars/all/projects.yml`. The existing `controller_projects`
  role (already run by `dispatch` on every apply, before `controller_job_templates`) then
  forces the sync with no extra task, role, or collection. **Not yet confirmed** via a live
  rerun.

## Resolved (2026-07-20)

- **Azure snapshot name exceeded the 80-character limit.** `snapshot_azure.yml` built the
  snapshot `name` from `snapshot_name_prefix` + hostname + the managed disk's own basename
  (which includes Azure's auto-generated `<vmname>_OsDisk_1_<32-char-guid>` suffix) + a
  date/time suffix. For the test VM this produced a 100-character name, over Azure's
  80-character limit for snapshot resource names, so the Azure API rejected every disk with
  `InvalidParameter: The value of parameter snapshot.name is invalid` (400). Fixed by replacing
  the disk basename in the name with a short, deterministic role label (`osdisk` for the OS
  disk, `datadiskN` for data disks) derived from the loop position via a new
  `loop_control.index_var`, instead of the disk's actual Azure-generated name. The resulting
  name for the test VM is 49 characters. **Not yet confirmed** via a live rerun.

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
