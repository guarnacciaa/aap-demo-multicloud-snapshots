# Demo procedures

## 1. Apply configuration as code

```bash
ansible-playbook playbooks/aap_config.yml -i inventory.yml --vault-id @prompt
```

Expected: AAP organization, project, inventories, credentials, job templates, and workflow are created or updated.

## 2. Run from AAP UI

1. Open **Templates** → **WF - Multicloud snapshot and retention**.
2. Launch the workflow (no survey required if extra vars are set on job templates).
3. Monitor nodes: Azure snapshot → AWS snapshot → Verify → Cleanup (optional).

## 3. Run individual job templates

### Azure snapshot

```bash
ansible-playbook playbooks/demo/snapshot_azure.yml \
  -e target_hostname=azure-demo-vm01 \
  -e azure_resource_group=my-rg \
  --vault-id @prompt
```

### AWS snapshot

```bash
ansible-playbook playbooks/demo/snapshot_aws.yml \
  -e target_hostname=aws-demo-vm01 \
  -e aws_region=us-east-1 \
  --vault-id @prompt
```

### Verify

```bash
ansible-playbook playbooks/demo/snapshot_verify.yml \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1
```

### Cleanup (optional)

```bash
ansible-playbook playbooks/demo/snapshot_cleanup.yml \
  -e enable_snapshot_cleanup=true \
  -e snapshot_retention_days=30 \
  -e azure_resource_group=my-rg \
  -e aws_region=us-east-1
```

## 4. Teardown

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```
