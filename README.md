# Multicloud VM Snapshot and Retention Demo

![Status](https://img.shields.io/badge/Status-Ready-brightgreen)
![Red Hat Ansible Automation Platform](https://img.shields.io/badge/AAP-2.6-red)
![Configuration as Code](https://img.shields.io/badge/CasC-infra.aap_configuration-blue)
![Azure](https://img.shields.io/badge/Azure-azcollection-0078d4)
![AWS](https://img.shields.io/badge/AWS-amazon.aws-ff9900)

## Introduction

This demo automates the **full lifecycle** of a multicloud VM snapshot and retention PoC: **infrastructure provisioning**, **disk snapshot creation and retention**, and **complete teardown**. No pre-existing cloud infrastructure is required; every resource (VMs, networking, security groups) is created and destroyed by the automation.

A single AAP workflow provisions Azure and AWS VMs, while a second workflow snapshots those VMs, verifies outcomes, and optionally purges obsolete snapshots. A third workflow tears everything down for clean post-demo cleanup.

**Audience:** Platform engineers evaluating AAP for multicloud backup operations with Configuration as Code.

## How to run the demo

| Phase | AAP object | Purpose |
|---|---|---|
| 1 - Setup | `WF - Demo setup (provision infrastructure)` | Create Azure VM, AWS EC2, and all networking |
| 2 - Demo | `WF - Multicloud snapshot and retention` | Snapshot VMs, verify, optionally clean old snapshots |
| 3 - Teardown | `WF - Demo teardown (destroy infrastructure)` | Remove snapshots and all provisioned resources |

Phases 1 and 3 only exist when `demo_manage_infrastructure: true` (default); see [Deployment modes](#deployment-modes) for the customer/PoC mode that runs phase 2 only, against pre-existing infrastructure.

1. Apply CasC: see [Quick start](#quick-start).
2. In AAP, open **Templates** and launch workflows in order: Setup, Demo, Teardown.
3. Individual job templates can also be launched standalone for targeted operations.

## Environment / prerequisites

- AAP 2.6+, Azure subscription, AWS account, cross-cloud API connectivity from AAP.
- Azure Service Principal with permissions to manage resource groups, networking, VMs, disks, and snapshots.
- AWS IAM user or role with EC2, VPC, and EBS permissions (create/describe/delete).
- A RHEL 9 AMI ID for your chosen AWS region.
- See [docs/setup.md](docs/setup.md) for detailed credential and EE setup.

## Deployment modes

`demo_manage_infrastructure` in `demo_variables.yml` (default `true`) controls how much of the AAP catalog this CasC creates:

| Mode | `demo_manage_infrastructure` | AAP objects created | Use when |
|---|---|---|---|
| Lab / dev | `true` (default) | Full lifecycle: `Provision - *`, `Update - Multicloud inventory hosts`, `Deprovision - *`, `WF - Demo setup`, `WF - Demo teardown`, plus all snapshot objects | You are running the demo yourself and want AAP to create and destroy the VMs and networking |
| Customer / PoC | `false` | Only `Snapshot - Azure/AWS by hostname`, `Snapshot - Verify`, `Snapshot - Cleanup (optional)`, and `WF - Multicloud snapshot and retention` | The customer already provides the VMs and networking; no provisioning or teardown object is created in AAP, removing any risk of accidentally launching a job that creates duplicate resources or deletes the customer's existing networking |

In customer mode, set `azure_vm_hostname` / `azure_resource_group` and `aws_ec2_hostname` / `aws_region` to the customer's existing VM/instance, and scope the Azure/AWS credentials down to read + snapshot permissions only (see [docs/setup.md](docs/setup.md)).

`group_vars/all/demo_variables.yml.example` marks every variable with `[ALWAYS REQUIRED]` or `[LAB/DEV ONLY]` banners so you can see at a glance what customer/PoC mode needs. `playbooks/aap_config.yml` and `playbooks/verify.yml` enforce this: the `[LAB/DEV ONLY]` variables are only validated when `demo_manage_infrastructure: true`, so leaving them at their example defaults never blocks a customer/PoC deployment.

## Quick start

```bash
cd aap-demo-multicloud-snapshots
ansible-galaxy collection install -r collections/requirements.yml -p collections
cp ansible.cfg.example ansible.cfg
cp group_vars/all/demo_variables.yml.example group_vars/all/demo_variables.yml
cp vault.yml.example vault.yml && ansible-vault encrypt vault.yml
```

Edit `group_vars/all/demo_variables.yml` with your AAP URL, Git repository URL (`demo_project_scm_url`), Azure/AWS identifiers, VM settings, and networking configuration. Edit `vault.yml` with secrets.

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

## Reset AAP configuration

To remove all CasC objects created by this demo (organization, project, templates, inventories, credentials, execution environment):

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```

Re-apply CasC with `aap_config.yml` when you want to deploy again.

## Scenario overview

| Challenge | AAP approach |
|---|---|
| No pre-existing infrastructure | Provisioning playbooks create VMs and networking from scratch |
| Different APIs per cloud | Certified `azure.azcollection` and `amazon.aws` modules |
| Consistent operational workflow | Workflow templates for setup, snapshot, and teardown |
| Reproducible platform config | `infra.aap_configuration` CasC for all AAP objects |
| Multicloud inventory UX (UC4) | Parent `Demo-Multicloud` inventory with Azure/AWS children |
| Clean post-demo state | Teardown workflow destroys all resources including snapshots |

## Architecture

```
                         AAP Controller
                    +---------------------+
                    |                     |
          +---------+---------+  +--------+---------+
          | WF - Demo setup   |  | WF - Snapshot    |
          | (provision infra) |  | and retention    |
          +---+----------+----+  +---+----+----+----+
              |          |           |    |    |
     +--------+    +-----+---+  +---+  +-+-+  +------+
     | Azure  |    | AWS EC2 |  | Az | |AWS|  |Verify|
     | VM     |    |         |  |snap| |snap|  |      |
     +--------+    +---------+  +----+ +----+  +------+

          +---------------------+
          | WF - Demo teardown  |
          | (destroy infra)     |
          +---+----------+------+
              |          |
     +--------+--+  +---+--------+
     | Cleanup    |  | Cleanup    |
     | snapshots  |  | snapshots  |
     +-----+------+  +------+----+
           |                 |
     +-----+------+  +------+----+
     | Deprovision|  |Deprovision|
     | Azure VM   |  | AWS EC2   |
     +------------+  +-----------+
```

## Multicloud inventory UX (UC4)

```
Demo-Multicloud
+-- Azure-Resources
|   +-- azure_vms (Azure demo VM)
+-- AWS-Resources
    +-- aws_vms (AWS demo VM)
```

CasC in `group_vars/all/inventories.yml` defines the hierarchy. Re-running `playbooks/aap_config.yml` reproduces inventory structure from Git.

## Repository map

| Path | Purpose |
|---|---|
| `playbooks/demo/provision_*.yml` | Infrastructure provisioning (Azure VM, AWS EC2) |
| `playbooks/demo/snapshot_*.yml` | Snapshot, verify, and cleanup automation |
| `playbooks/demo/deprovision_*.yml` | Infrastructure teardown (Azure, AWS) |
| `playbooks/aap_config.yml` | Apply CasC to AAP |
| `playbooks/aap_cleanup.yml` | Remove all CasC objects from AAP |
| `group_vars/all/` | CasC variable definitions |
| `group_vars/all/job_templates_infra.yml`, `group_vars/all/workflow_templates_infra.yml` | Provisioning/teardown job and workflow templates; merged in only when `demo_manage_infrastructure: true` (see [Deployment modes](#deployment-modes)) |
| `context/execution-environment.yml` | Optional custom EE |
| `docs/` | Setup, procedures, verification |

## Job templates

Tables below list every job template the demo defines. The **Infrastructure provisioning** and **Infrastructure teardown** templates are only created in AAP when `demo_manage_infrastructure: true` (see [Deployment modes](#deployment-modes)); the **Snapshot operations** templates are always created.

### Infrastructure provisioning

| Job template | Playbook | Credentials |
|---|---|---|
| Provision - Azure VM | `playbooks/demo/provision_azure_vm.yml` | Azure SP |
| Provision - AWS EC2 | `playbooks/demo/provision_aws_ec2.yml` | AWS IAM |
| Update - Multicloud inventory hosts | `playbooks/demo/update_inventory_hosts.yml` | Azure SP, AWS IAM |

### Snapshot operations

| Job template | Playbook | Credentials |
|---|---|---|
| Snapshot - Azure by hostname | `playbooks/demo/snapshot_azure.yml` | Azure SP |
| Snapshot - AWS by hostname | `playbooks/demo/snapshot_aws.yml` | AWS IAM |
| Snapshot - Verify | `playbooks/demo/snapshot_verify.yml` | Azure SP, AWS IAM |
| Snapshot - Cleanup (optional) | `playbooks/demo/snapshot_cleanup.yml` | Azure SP, AWS IAM |

### Infrastructure teardown

| Job template | Playbook | Credentials |
|---|---|---|
| Deprovision - Azure VM | `playbooks/demo/deprovision_azure_vm.yml` | Azure SP |
| Deprovision - AWS EC2 | `playbooks/demo/deprovision_aws_ec2.yml` | AWS IAM |

## Workflow templates

`WF - Demo setup` and `WF - Demo teardown` are only created when `demo_manage_infrastructure: true` (see [Deployment modes](#deployment-modes)); `WF - Multicloud snapshot and retention` is always created.

| Workflow | Nodes | Purpose | Mode |
|---|---|---|---|
| WF - Demo setup (provision infrastructure) | Provision Azure VM + Provision AWS EC2 (parallel) → Update - Multicloud inventory hosts | Create all demo infrastructure and populate inventories | Lab/dev only |
| WF - Multicloud snapshot and retention | Azure snapshot -> AWS snapshot -> Verify -> Cleanup | Run the snapshot demo | Always |
| WF - Demo teardown (destroy infrastructure) | Cleanup snapshots -> Deprovision Azure + Deprovision AWS (parallel) | Remove everything | Lab/dev only |

## Collections

| Collection | Tier | Purpose |
|---|---|---|
| infra.aap_configuration | validated | AAP Configuration as Code |
| azure.azcollection | certified | Azure networking, VM, and snapshot modules |
| amazon.aws | certified | VPC, EC2, and EBS snapshot modules |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Azure provisioning fails on public IP | Quota exceeded or region unavailable | Change `azure_location` or request quota increase |
| AWS EC2 launch fails | Invalid AMI ID for region | Set `aws_ami_id` to a valid RHEL 9 AMI for `aws_region` |
| Snapshot playbook finds 0 VMs | Infrastructure not provisioned | Run the setup workflow first |
| Deprovision skips network resources | VPC/VNet already deleted | Safe to ignore; teardown is idempotent |
| Cross-cloud API timeout | AAP cannot reach AWS from Azure | Check network connectivity and security group rules |
| `image not known` on `podman push` | Local image not tagged for Hub registry | Tag first: `podman tag localhost/ee-multicloud-snapshots:latest <aap_host>/ee-multicloud-snapshots:latest` |
| TLS error on `podman login` to Hub | Self-signed certificate on growth deployment | Use `--tls-verify=false` on login and push; see [docs/setup.md](docs/setup.md) |
| TLS error when Controller pulls EE | Missing Container Registry credential or Verify SSL enabled | Re-run CasC; attach `PAH Container Registry` credential with Verify SSL disabled |
