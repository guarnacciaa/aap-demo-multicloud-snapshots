# Environment setup

## Requirements

| Component | Version |
|---|---|
| Red Hat Ansible Automation Platform | 2.6 or later |
| Ansible Core | 2.16+ (bundled with AAP) |
| Azure subscription | VM with managed disks |
| AWS account | EC2 with EBS volumes |
| Network | AAP (Azure) must reach AWS API endpoints |

## Customer-provided prerequisites

- One Azure VM hostname and one AWS EC2 hostname (Name tag on EC2).
- Azure Service Principal with permissions to read VMs/disks and create/delete snapshots.
- AWS IAM user or role with `ec2:DescribeInstances`, `ec2:CreateSnapshot`, `ec2:DescribeSnapshots`, `ec2:DeleteSnapshot`.
- Azure resource group containing the demo VM.

## Install collections

```bash
cd artifacts/demos/aap-demo-multicloud-snapshots
ansible-galaxy collection install -r collections/requirements.yml -p collections
cp ansible.cfg.example ansible.cfg
```

## Configure variables and secrets

```bash
cp group_vars/all/demo_variables.yml.example group_vars/all/demo_variables.yml
cp vault.yml.example vault.yml
ansible-vault encrypt vault.yml
```

Edit `group_vars/all/demo_variables.yml` with AAP URL, Azure/AWS identifiers, and hostnames.

## Vault

Store `vault_controller_password`, `vault_azure_client_secret`, and `vault_aws_secret_key` in `vault.yml`.

## Custom Execution Environment (optional)

If your platform EE lacks cloud SDKs:

```bash
cd context
ansible-builder build -f execution-environment.yml -t ee-multicloud-snapshots:latest
```

Push the image to your registry and update `group_vars/all/execution_environments.yml`.

## Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

## Multicloud inventory (UC4)

This artifact provisions a parent inventory `Demo-Multicloud` with child inventories `Azure-Resources` and `AWS-Resources`. Hosts and groups are defined in CasC (`hosts.yml`, `groups.yml`). Update host IP placeholders before applying CasC.
