# Verification

## Automated smoke test

```bash
ansible-playbook playbooks/verify.yml
```

Passing run confirms demo variables and `infra.aap_configuration` collection are available.

## Functional verification

1. Launch **WF - Multicloud snapshot and retention** from AAP.
2. Confirm each workflow node completes successfully.
3. In Azure Portal, verify managed disk snapshots with tag `Demo=multicloud-snapshots`.
4. In AWS Console, verify EBS snapshots with tag `Demo=multicloud-snapshots` and `state=completed`.

## Manual checks

```bash
# Azure (Azure CLI)
az snapshot list --resource-group <rg> --query "[?tags.Demo=='multicloud-snapshots']"

# AWS (AWS CLI)
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=tag:Demo,Values=multicloud-snapshots"
```

## References

- [azure.azcollection.azure_rm_snapshot](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_snapshot_module.html)
- [amazon.aws.ec2_snapshot](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_snapshot_module.html)
- [infra.aap_configuration](https://docs.ansible.com/ansible/latest/collections/infra/aap_configuration/)
