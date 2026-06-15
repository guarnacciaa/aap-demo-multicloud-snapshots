# Testing log

Tracks testing progress for this demo. Update after each session. For procedural verification steps, see [verification.md](verification.md).

**Status values:** `Not tested` · `Pass` · `Partial` · `Fail` · `Blocked`

## Status summary

### Entry playbooks

| Component | Status | Last tested | Notes |
|---|---|---|---|
| CasC apply (`aap_config.yml`) | Pass | 2026-06-12 | |
| CasC cleanup (`aap_cleanup.yml`) | Pass | 2026-06-12 | |
| Smoke test (`verify.yml`) | Pass | 2026-06-15 | All 7 tasks ok on bastion: variables, collections (infra.aap_configuration, azure.azcollection, amazon.aws) |

### Execution environments

| Component | Status | Last tested | Notes |
|---|---|---|---|
| EE build and push (azure.azcollection + amazon.aws) | Pass | 2026-06-12 | |

### Inventories

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Azure-Resources (source) | Pass | 2026-06-12 | |
| AWS-Resources (source) | Pass | 2026-06-12 | |
| Demo-Multicloud (constructed) | Pass | 2026-06-12 | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Provision - Azure VM | Pass | 2026-06-12 | |
| Provision - AWS EC2 | Pass | 2026-06-12 | Deprecated `network` param replaced with `network_interfaces` |
| Update - Multicloud inventory hosts | Pass | 2026-06-13 | Ran as part of WF - Demo setup |
| Snapshot - Azure by hostname | Pass | 2026-06-13 | Fixed: azure_rm_virtualmachine_info omits storage_profile; switched to azure_rm_resource_info (provider: Compute). set_stats key was state.id → id. |
| Snapshot - AWS by hostname | Pass | 2026-06-15 | Confirmed via Snapshot - Verify output (3 completed snapshots found) |
| Snapshot - Verify | Pass | 2026-06-15 | Fixed: missing extra_vars on job template; azure_rm_snapshot_info replaced with azure_rm_resource_info (api_version: 2026-03-02) |
| Snapshot - Cleanup (optional) | Pass | 2026-06-15 | Count mode: kept 1 AWS snapshot, deleted 4; Azure listing fixed (azure_rm_resource_info api_version 2026-03-02) |
| Deprovision - Azure VM | Pass | 2026-06-12 | |
| Deprovision - AWS EC2 | Pass | 2026-06-12 | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Pass | 2026-06-13 | |
| WF - Multicloud snapshot and retention | Pass | 2026-06-15 | All nodes confirmed: snapshot Azure, snapshot AWS, verify, cleanup (count mode) |
| WF - Demo teardown (destroy infrastructure) | Pass | 2026-06-13 | |

## Open issues

- None
