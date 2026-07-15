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
| Provision - Azure VM | Not tested | — | Lab/dev mode only |
| Provision - AWS EC2 | Not tested | — | Lab/dev mode only |
| Update - Multicloud inventory hosts | Not tested | — | Lab/dev mode only |
| Snapshot - Connectivity check (dry run) | Not tested | — | Read-only credential/connectivity check |
| Snapshot - Preview (dry run) | Not tested | — | Read-only pre-flight check |
| Snapshot - Azure by hostname | Not tested | — | |
| Snapshot - AWS by hostname | Not tested | — | |
| Snapshot - Verify | Not tested | — | |
| Snapshot - Cleanup preview (dry run) | Not tested | — | Read-only retention policy preview |
| Snapshot - Cleanup (optional) | Not tested | — | |
| Snapshot - Inventory report | Not tested | — | Standalone audit tool, not in the workflow |
| Deprovision - Azure VM | Not tested | — | Lab/dev mode only |
| Deprovision - AWS EC2 | Not tested | — | Lab/dev mode only |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Not tested | — | Lab/dev mode only |
| WF - Multicloud snapshot and retention | Not tested | — | Now starts with connectivity check -> preview, and includes a cleanup preview node before cleanup |
| WF - Demo teardown (destroy infrastructure) | Not tested | — | Lab/dev mode only |

## Open issues

- None
