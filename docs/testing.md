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
| _(none — this demo uses no custom roles)_ | — | — | |

### Execution environments

| Component | Status | Last tested | Notes |
|---|---|---|---|
| EE build and push (`ee-multicloud-snapshots`) | Not tested | — | |

### Inventories

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Azure-Resources | Not tested | — | |
| AWS-Resources | Not tested | — | |
| Demo-Multicloud (constructed) | Not tested | — | |

### Job templates

| Component | Status | Last tested | Notes |
|---|---|---|---|
| Provision - Azure VM | Not tested | — | |
| Provision - AWS EC2 | Not tested | — | |
| Update - Multicloud inventory hosts | Not tested | — | |
| Snapshot - Azure by hostname | Not tested | — | |
| Snapshot - AWS by hostname | Not tested | — | |
| Snapshot - Verify | Not tested | — | |
| Snapshot - Cleanup (optional) | Not tested | — | |
| Deprovision - Azure VM | Not tested | — | |
| Deprovision - AWS EC2 | Not tested | — | |

### Workflows

| Component | Status | Last tested | Notes |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Not tested | — | |
| WF - Multicloud snapshot and retention | Not tested | — | |
| WF - Demo teardown (destroy infrastructure) | Not tested | — | |

## Open issues

- `azure_auth_mode: msi` (Managed Identity authentication, added to support running Azure tasks without a stored client secret) is implemented in every `azure_rm_*` task via `auth_source` but has not been exercised end-to-end. It requires an AAP execution node/EE container running on an Azure resource with an identity attached, which was not available at implementation time. The default `service_principal` path is unaffected (same behavior as before this change) but is also still `Not tested` per the table above.
