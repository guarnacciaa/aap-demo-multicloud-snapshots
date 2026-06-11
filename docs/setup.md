# Environment setup

## Requirements

| Component | Version / detail |
|---|---|
| Red Hat Ansible Automation Platform | 2.6 or later |
| Ansible Core | 2.16+ (bundled with AAP) |
| Azure subscription | With permissions to create resource groups, VNets, NSGs, public IPs, NICs, and VMs |
| AWS account | With permissions to create VPCs, subnets, IGWs, route tables, security groups, key pairs, and EC2 instances |
| Network | AAP must reach both Azure and AWS API endpoints |

## Azure prerequisites

- **Service Principal** with the following permissions (Contributor role on the subscription or resource group scope is sufficient):
  - Resource group: create, read, delete
  - Virtual network and subnet: create, read, delete
  - Network security group: create, read, delete
  - Public IP address: create, read, delete
  - Network interface: create, read, delete
  - Virtual machine: create, read, delete
  - Managed disk and snapshot: create, read, delete
- Note the **subscription ID**, **tenant ID**, **client ID**, and **client secret**.

## AWS prerequisites

- **IAM user or role** with the following permissions:
  - `ec2:CreateVpc`, `ec2:DeleteVpc`, `ec2:DescribeVpcs`
  - `ec2:CreateSubnet`, `ec2:DeleteSubnet`, `ec2:DescribeSubnets`
  - `ec2:CreateInternetGateway`, `ec2:DeleteInternetGateway`, `ec2:AttachInternetGateway`, `ec2:DetachInternetGateway`
  - `ec2:CreateRouteTable`, `ec2:DeleteRouteTable`, `ec2:CreateRoute`, `ec2:AssociateRouteTable`
  - `ec2:CreateSecurityGroup`, `ec2:DeleteSecurityGroup`, `ec2:AuthorizeSecurityGroupIngress`
  - `ec2:CreateKeyPair`, `ec2:DeleteKeyPair`, `ec2:ImportKeyPair`
  - `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:DescribeInstances`
  - `ec2:CreateSnapshot`, `ec2:DeleteSnapshot`, `ec2:DescribeSnapshots`
  - `ec2:CreateTags`
- A **RHEL 9 AMI ID** for your target region. Find yours at [Red Hat Cloud Access](https://access.redhat.com/solutions/15356) or via the AWS EC2 console.
- Note the **access key ID** and **secret access key**.

## Install collections

```bash
cd aap-demo-multicloud-snapshots
ansible-galaxy collection install -r collections/requirements.yml -p collections
cp ansible.cfg.example ansible.cfg
```

## Configure variables and secrets

```bash
cp group_vars/all/demo_variables.yml.example group_vars/all/demo_variables.yml
cp vault.yml.example vault.yml
ansible-vault encrypt vault.yml
```

Edit `group_vars/all/demo_variables.yml` with:

- AAP URL and admin credentials
- Azure subscription, tenant, client, resource group, location, VM size, and networking names
- AWS region, instance type, AMI ID, key pair name, and networking names

## Vault

Store these secrets in `vault.yml`:

| Vault variable | Purpose |
|---|---|
| `vault_controller_password` | AAP admin password |
| `vault_azure_client_secret` | Azure Service Principal client secret |
| `vault_azure_vm_admin_password` | Azure VM admin password (set during provisioning) |
| `vault_aws_secret_key` | AWS IAM secret access key |

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

This creates or updates all AAP objects: organization, project, inventories, credentials, job templates (provisioning, snapshot, and teardown), and all three workflow templates.

## Multicloud inventory (UC4)

This artifact provisions a parent inventory `Demo-Multicloud` with child inventories `Azure-Resources` and `AWS-Resources`. Hosts and groups are defined in CasC (`hosts.yml`, `groups.yml`). After provisioning the VMs, update the host IP placeholders in `hosts.yml` if you want host-level operations beyond cloud API calls.
