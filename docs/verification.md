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

## Snapshot functional verification

1. Launch **WF - Multicloud snapshot and retention** from AAP.
2. Confirm each workflow node completes successfully.
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
