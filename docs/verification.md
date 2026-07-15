# Verification

## Automated smoke test

```bash
ansible-playbook playbooks/verify.yml
```

Passing run confirms demo variables and `infra.aap_configuration` collection are available.

## Infrastructure provisioning verification

After running **WF - Demo setup (provision infrastructure)**:

1. In AAP, confirm both workflow nodes completed successfully.
2. In Azure Portal, verify the VM exists with tag `Demo=multicloud-snapshots` in the configured resource group.
3. In AWS Console, verify the EC2 instance is running with tag `Demo=multicloud-snapshots`.

Manual checks:

```bash
# Azure (Azure CLI)
az vm show --resource-group <rg> --name <azure-vm-hostname> --query "provisioningState"

# AWS (AWS CLI)
aws ec2 describe-instances --filters "Name=tag:Name,Values=<aws-ec2-hostname>" \
  --query "Reservations[].Instances[].State.Name"
```

## Pre-flight check verification

1. Launch **Snapshot - Connectivity check (dry run)** standalone, or check the first node of **WF - Multicloud snapshot and retention**. Confirm the job output reports a successful Azure subscription lookup and AWS caller identity. A failure here means the Azure SP/Managed Identity or AWS IAM credential itself is misconfigured, independent of any target VM.
2. Launch **Snapshot - Preview (dry run)** standalone, or check the second node of the workflow. Confirm the job output reports exactly one Azure VM and one AWS EC2 instance, with the disk/volume list matching what you expect (compare against the Azure Portal / AWS Console).
3. If either job fails with "found 0" or "found N > 1", fix `azure_resource_group` / `azure_vm_hostname` / `aws_ec2_hostname` / `aws_region` in `demo_variables.yml` before proceeding — no snapshot is created when these checks fail.
4. Launch **Snapshot - Cleanup preview (dry run)** to see which existing snapshots (if any) the current retention policy would delete; confirm the list matches your expectations before setting `enable_snapshot_cleanup: true`.
5. Launch **Snapshot - Inventory report** at any time to list every demo snapshot currently in Azure/AWS with age and size — useful as a baseline before and after a demo run.

## Snapshot functional verification

1. Launch **WF - Multicloud snapshot and retention** from AAP.
2. Confirm each workflow node completes successfully, starting with the connectivity check and preview (dry run) nodes.
3. In Azure Portal, verify managed disk snapshots with tag `Demo=multicloud-snapshots`.
4. In AWS Console, verify EBS snapshots with tag `Demo=multicloud-snapshots` and `state=completed`.

Manual checks:

```bash
# Azure (Azure CLI)
az snapshot list --resource-group <rg> --query "[?tags.Demo=='multicloud-snapshots']"

# AWS (AWS CLI)
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=tag:Demo,Values=multicloud-snapshots"
```

## Teardown verification

After running **WF - Demo teardown (destroy infrastructure)**:

1. Confirm all workflow nodes completed successfully.
2. Verify snapshots are removed (see snapshot verification commands above).
3. Verify Azure VM and networking no longer exist in the resource group.
4. Verify AWS EC2 instance is terminated and VPC is deleted.

```bash
# Azure - should return empty or not found
az vm show --resource-group <rg> --name <azure-vm-hostname> 2>&1 | grep -c "not found"

# AWS - should show terminated or empty
aws ec2 describe-instances --filters "Name=tag:Name,Values=<aws-ec2-hostname>" \
  --query "Reservations[].Instances[?State.Name!='terminated']"
```

## References

- [azure.azcollection.azure_rm_virtualmachine](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_virtualmachine_module.html)
- [azure.azcollection.azure_rm_snapshot](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_snapshot_module.html)
- [amazon.aws.ec2_instance](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html)
- [amazon.aws.ec2_snapshot](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_snapshot_module.html)
- [infra.aap_configuration](https://docs.ansible.com/ansible/latest/collections/infra/aap_configuration/)
