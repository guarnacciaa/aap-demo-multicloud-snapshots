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

If your platform EE lacks cloud SDKs, build a custom image from `context/execution-environment.yml`. The definition uses the AAP 2.6 `ee-minimal-rhel9` base image from `registry.redhat.io` (not the legacy `quay.io/ansible/ansible-runner` default).

**Prerequisites**

1. Log in to the Red Hat container registry and pull the base image:

   ```bash
   podman login registry.redhat.io
   podman pull registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:latest
   ```

2. Copy and configure `ansible.cfg` at the **artifact root** (required; the build copies `../ansible.cfg` into the image build context):

   ```bash
   cd ~/aap-demo-multicloud-snapshots
   cp ansible.cfg.example ansible.cfg
   # Replace <token> with your Automation Hub offline token in BOTH
   # [galaxy_server.automation_hub_certified] and
   # [galaxy_server.automation_hub_validated].
   ```

   Verify the token works before building the EE:

   ```bash
   ansible-galaxy collection install -r collections/requirements.yml -p collections
   ```

   If this command returns `HTTP Code: 401`, fix `ansible.cfg` before running `ansible-builder`.

**Build and register**

Run the build from the artifact root so paths to `ansible.cfg` and the build context resolve correctly. Remove any previous build output first:

```bash
cd ~/aap-demo-multicloud-snapshots
rm -rf context/_build context/Containerfile

ansible-builder build \
  -f context/execution-environment.yml \
  -t ee-multicloud-snapshots:latest \
  context
```

Push the image to your registry and update `group_vars/all/execution_environments.yml` with the full image reference (registry, name, and tag).

**Troubleshooting**

| Symptom | Likely cause | Fix |
|---|---|---|
| `HTTP Code: 401, Message: Unauthorized` during `ansible-galaxy collection install` in the build | Missing or placeholder token in `../ansible.cfg`, or `ansible.cfg` not created at artifact root | Create `ansible.cfg` from the example and set a valid Hub token; confirm with the local `ansible-galaxy` command above |
| `explicit_requirement_infra.aap_configuration ... 401` | Old `context/requirements.yml` with per-collection `source:` URLs or validated collections in the EE | Use the current `context/requirements.yml` (certified cloud collections only; no `source:` keys) |
| Warning about `quay.io/ansible/ansible-runner` | Missing `images.base_image` in an old definition file | Use the current `context/execution-environment.yml` with `ee-minimal-rhel9` |
| `/usr/bin/python3: No module named pip` in the builder stage | Explicit `python:` / `python3-pip` entries forced a UBI Python stack into the builder | Use the current galaxy-only `context/execution-environment.yml`; collections supply pip deps |
| `ModuleNotFoundError: No module named 'yaml'` in `introspect.py` | Same root cause: microdnf `python3-pip` in the builder stage conflicts with `ee-minimal-rhel9` | Remove custom `prepend_builder` / system pip packages; rebuild from a clean `context/_build` directory |

## Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

This creates or updates all AAP objects: organization, project, inventories, credentials, job templates (provisioning, snapshot, and teardown), and all three workflow templates.

## Multicloud inventory (UC4)

This artifact provisions a parent inventory `Demo-Multicloud` with child inventories `Azure-Resources` and `AWS-Resources`. Hosts and groups are defined in CasC (`hosts.yml`, `groups.yml`). After provisioning the VMs, update the host IP placeholders in `hosts.yml` if you want host-level operations beyond cloud API calls.
