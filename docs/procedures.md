# Demo procedures

## 1. Apply configuration as code

```bash
ansible-playbook playbooks/aap_config.yml -i inventory.yml --vault-id @prompt
```

Expected: AAP organization, project, inventories, credentials, job templates, and all three workflows are created or updated.

## 2. Provision demo infrastructure

### From AAP UI

1. Open **Templates** -> **WF - Demo setup (provision infrastructure)**.
2. Launch **WF - Demo setup (provision infrastructure)**. Azure and AWS provisioning run in parallel, then **Update - Multicloud inventory hosts** refreshes AAP host variables.
3. Monitor all nodes until they complete. Note the public IPs from the job output.

### From CLI (ad hoc)

Azure VM:

```bash
ansible-playbook playbooks/demo/provision_azure_vm.yml \
  --vault-id @prompt
```

AWS EC2:

```bash
ansible-playbook playbooks/demo/provision_aws_ec2.yml \
  -e aws_ami_id=ami-0123456789abcdef0 \
  --vault-id @prompt
```

## 3. Run the snapshot demo

### From AAP UI

1. Open **Templates** -> **WF - Multicloud snapshot and retention**.
2. Launch the workflow (no survey required if extra vars are set on job templates).
3. Monitor nodes: Connectivity check -> Preview (dry run) -> Azure snapshot -> AWS snapshot -> Verify -> Cleanup preview (dry run) -> Cleanup (optional). The four dry-run nodes are read-only and fail fast (or simply report) before any snapshot is created or deleted.

### Pre-flight checks (optional, standalone)

Each of the following can be launched on its own — as its own job template, or from the CLI — without running the full workflow:

Connectivity check (validates Azure/AWS credentials, no target VM needed):

```bash
ansible-playbook playbooks/demo/precheck_connectivity.yml \
  -e azure_subscription_id=00000000-0000-0000-0000-000000000000 \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

Preview target VMs and disks (see exactly which VM/instance and disks/volumes will be snapshotted):

```bash
ansible-playbook playbooks/demo/snapshot_preview.yml \
  -e azure_resource_group=my-rg \
  -e azure_vm_hostname=azure-demo-vm01 \
  -e aws_ec2_hostname=aws-demo-vm01 \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

Cleanup preview (see which existing snapshots the retention policy would delete, without deleting anything):

```bash
ansible-playbook playbooks/demo/snapshot_cleanup_preview.yml \
  -e snapshot_retention_days=30 \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

Snapshot inventory report (list every demo snapshot currently in Azure/AWS, any time):

```bash
ansible-playbook playbooks/demo/snapshot_inventory.yml \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

These are the same checks that run automatically inside **WF - Multicloud snapshot and retention**; run them standalone (or as their AAP job templates) whenever you only want a status check without launching the full workflow.

### Run individual snapshot templates

Azure snapshot:

```bash
ansible-playbook playbooks/demo/snapshot_azure.yml \
  -e target_hostname=azure-demo-vm01 \
  -e azure_resource_group=my-rg \
  --vault-id @prompt
```

AWS snapshot:

```bash
ansible-playbook playbooks/demo/snapshot_aws.yml \
  -e target_hostname=aws-demo-vm01 \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

Verify:

```bash
ansible-playbook playbooks/demo/snapshot_verify.yml \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1
```

Cleanup (optional):

```bash
ansible-playbook playbooks/demo/snapshot_cleanup.yml \
  -e enable_snapshot_cleanup=true \
  -e snapshot_retention_days=30 \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1
```

## 4. Tear down demo infrastructure

### From AAP UI (recommended)

1. Open **Templates** -> **WF - Demo teardown (destroy infrastructure)**.
2. Launch the workflow.
3. The workflow first cleans up all demo snapshots, then deprovisions Azure and AWS resources in parallel.

### From CLI (ad hoc)

Deprovision Azure VM and networking:

```bash
ansible-playbook playbooks/demo/deprovision_azure_vm.yml \
  --vault-id @prompt
```

To also delete the Azure resource group:

```bash
ansible-playbook playbooks/demo/deprovision_azure_vm.yml \
  -e azure_delete_resource_group=true \
  --vault-id @prompt
```

Deprovision AWS EC2 and networking:

```bash
ansible-playbook playbooks/demo/deprovision_aws_ec2.yml \
  --vault-id @prompt
```

To also delete the AWS key pair:

```bash
ansible-playbook playbooks/demo/deprovision_aws_ec2.yml \
  -e aws_delete_key_pair=true \
  --vault-id @prompt
```

## 5. Remove AAP objects (CasC cleanup)

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```
