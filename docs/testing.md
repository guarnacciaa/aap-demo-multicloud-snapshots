# Testing log

Tracks testing progress for this demo. Update after each session. For procedural verification steps, see [verification.md](verification.md).

**Status values:** `Not tested` · `Pass` · `Partial` · `Fail` · `Blocked`

## Status summary

### Entry playbooks

| Component | Status | Last tested | Notes |
|---|---|---|---|
| CasC apply (`aap_config.yml`) | Pass | 2026-06-12 | |
| CasC cleanup (`aap_cleanup.yml`) | Pass | 2026-06-12 | |
| Smoke test (`verify.yml`) | Not tested | — | |

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
| Update - Multicloud inventory hosts | Not tested | — | |
| Snapshot - Azure by hostname | Not tested | — | |
| Snapshot - AWS by hostname | Not tested | — | |
| Snapshot - Verify | Not tested | — | |
| Snapshot - Cleanup (optional) | Not tested | — | |
| Deprovision - Azure VM | Pass | 2026-06-12 | |
| Deprovision - AWS EC2 | Pass | 2026-06-12 | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Not tested | — | |
| WF - Multicloud snapshot and retention | Not tested | — | |
| WF - Demo teardown (destroy infrastructure) | Not tested | — | |

## Open issues

- None
