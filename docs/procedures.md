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
3. Monitor nodes: Azure snapshot -> AWS snapshot -> Verify -> Cleanup (optional).

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
