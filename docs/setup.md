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
- Git repository URL (`demo_project_scm_url`; see commented GitHub, Gitea, and GitLab options in the example file)
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

**Build**

Run the build from the artifact root so paths to `ansible.cfg` and the build context resolve correctly. Remove any previous build output first:

```bash
cd ~/aap-demo-multicloud-snapshots
rm -rf context/_build context/Containerfile

ansible-builder build \
  -f context/execution-environment.yml \
  -t ee-multicloud-snapshots:latest \
  context
```

After a successful build, the image is tagged locally as `localhost/ee-multicloud-snapshots:latest`. Confirm with `podman images`.

**Push to Private Automation Hub**

On a **container growth** (all-in-one) deployment, Private Automation Hub runs on the same host as the other AAP components. Push the built image to the Hub container registry using the Platform Gateway hostname or IP (the same host you use for `aap_hostname` in `demo_variables.yml`, without the `https://` prefix).

Replace `<aap_host>` below with that value (for example `192.168.178.225` or `aap.example.org`).

1. Tag the local image for the Hub registry (required before push):

   ```bash
   podman tag localhost/ee-multicloud-snapshots:latest \
     <aap_host>/ee-multicloud-snapshots:latest
   ```

2. Log in to the Hub registry with your AAP admin credentials:

   ```bash
   podman login --tls-verify=false <aap_host>
   ```

   Growth and lab deployments typically use a self-signed TLS certificate. Without `--tls-verify=false`, login fails with `x509: certificate signed by unknown authority`.

   Let Podman prompt for the password; do not pass it on the command line.

3. Push the tagged image:

   ```bash
   podman push --tls-verify=false <aap_host>/ee-multicloud-snapshots:latest
   ```

4. Verify in the AAP UI: **Automation Content** > **Execution Environments**. The image should appear in the container repository list.

   You can also copy the exact pull command from **Pull this image** on the execution environment detail page.

If login or push fails with a connection error, retry using the Hub NGINX port exposed by the containerized installer (default HTTPS port `8444`):

```bash
podman tag localhost/ee-multicloud-snapshots:latest \
  <aap_host>:8444/ee-multicloud-snapshots:latest

podman login --tls-verify=false <aap_host>:8444
podman push --tls-verify=false <aap_host>:8444/ee-multicloud-snapshots:latest
```

**Register in AAP**

Set `demo_execution_environment_image` in `group_vars/all/demo_variables.yml` to the full image reference you pushed (registry host, image name, and tag):

```yaml
demo_execution_environment_image: <aap_host>/ee-multicloud-snapshots:latest
```

CasC also creates a **Container Registry** credential (`PAH Container Registry`) with **Verify SSL** disabled and attaches it to the execution environment so the Controller can pull the image from a self-signed Hub registry.

Re-apply CasC:

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

Confirm in AAP: **Automation Execution** > **Infrastructure** > **Execution Environments** > `ee-multicloud-snapshots` shows the pushed image and the **PAH Container Registry** credential.

If job templates fail to pull the image, confirm **Verify SSL** is disabled on that credential and that `hub_registry_host` matches the registry host in `demo_execution_environment_image` (no `https://` prefix).

**Troubleshooting**

| Symptom | Likely cause | Fix |
|---|---|---|
| `HTTP Code: 401, Message: Unauthorized` during `ansible-galaxy collection install` in the build | Missing or placeholder token in `../ansible.cfg`, or `ansible.cfg` not created at artifact root | Create `ansible.cfg` from the example and set a valid Hub token; confirm with the local `ansible-galaxy` command above |
| `explicit_requirement_infra.aap_configuration ... 401` | Old `context/requirements.yml` with per-collection `source:` URLs or validated collections in the EE | Use the current `context/requirements.yml` (certified cloud collections only; no `source:` keys) |
| Warning about `quay.io/ansible/ansible-runner` | Missing `images.base_image` in an old definition file | Use the current `context/execution-environment.yml` with `ee-minimal-rhel9` |
| `/usr/bin/python3: No module named pip` in the builder stage | Explicit `python:` / `python3-pip` entries forced a UBI Python stack into the builder | Use the current galaxy-only `context/execution-environment.yml`; collections supply pip deps |
| `ModuleNotFoundError: No module named 'yaml'` in `introspect.py` | Same root cause: microdnf `python3-pip` in the builder stage conflicts with `ee-minimal-rhel9` | Remove custom `prepend_builder` / system pip packages; rebuild from a clean `context/_build` directory |
| `image not known` on `podman push` | Image not tagged for the Hub registry hostname | Run `podman tag localhost/ee-multicloud-snapshots:latest <aap_host>/ee-multicloud-snapshots:latest` before push |
| `x509: certificate signed by unknown authority` on `podman login` | Self-signed or internal CA certificate on the Hub registry | Use `podman login --tls-verify=false` and `podman push --tls-verify=false`; disable **Verify SSL** on the Container Registry credential in AAP |
| `401 Unauthorized` on `podman login` or push | Wrong AAP credentials or insufficient Hub permissions | Use a user with permission to create containers on Hub (for example the gateway admin); authenticate through the Platform Gateway |
| Connection refused on push | Wrong registry host or port | Confirm `<aap_host>` matches `aap_hostname`; retry with port `8444` as shown above |
| `couldn't resolve module/action 'azure.azcollection.*'` at job runtime | Job uses the placeholder EE (`quay.io/ansible/ansible-runner`) instead of the custom image | Set `demo_execution_environment_image` in `demo_variables.yml`, re-run `aap_config.yml`, and confirm the job template points to `ee-multicloud-snapshots` |
| `x509: certificate signed by unknown authority` when Controller pulls the EE | Hub registry uses a self-signed certificate and the EE has no registry credential with SSL verification disabled | Re-run CasC after setting `demo_execution_environment_image`; confirm `PAH Container Registry` credential exists with **Verify SSL** off and is linked to the EE |
| `Demo-Multicloud` inventory shows no hosts | Constructed inventory missing `input_inventories`, or hosts only in child inventories | Re-run `aap_config.yml`; confirm hosts under **Azure-Resources** / **AWS-Resources**; run **Update - Multicloud inventory hosts** after setup workflow |
| Sync task fails with HTTP 404 on `/api/v2/` | Legacy Controller API path on Platform Gateway | Use `ansible.controller` modules only; gateway path is `/api/controller/v2/` (AAP 2.5+) |
| Sync task: inventory source not found | Wrong source name (`Demo-Multicloud` vs auto-created name) | Source is `Auto-created source for: Demo-Multicloud`; sync task resolves it via API lookup |

## Apply CasC

```bash
ansible-playbook playbooks/aap_config.yml --vault-id @prompt
```

This creates or updates all AAP objects: organization, project, inventories, credentials, job templates (provisioning, snapshot, and teardown), and all three workflow templates.

## Reset AAP configuration

Remove all CasC objects created by this demo. Confirmation is required:

```bash
ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt
```

This deletes the organization, project, templates, inventories, credentials, and execution environment defined by this artifact. Re-run `aap_config.yml` to deploy again.

## Multicloud inventory (UC4)

This artifact provisions a parent inventory `Demo-Multicloud` with child inventories `Azure-Resources` and `AWS-Resources`. Hosts and groups are defined in CasC (`hosts.yml`, `groups.yml`).

**Constructed inventory:** `Demo-Multicloud` aggregates hosts from the two child inventories via `input_inventories`. The dispatch role does not always set `input_inventories` on first create; `aap_config.yml` reconciles this in a post-task. After CasC, you should see placeholder hosts in `Azure-Resources` and `AWS-Resources`; `Demo-Multicloud` lists the same hosts once wiring succeeds.

**After cloud provisioning:** the setup workflow runs **Update - Multicloud inventory hosts**, which sets `ansible_host` to each VM public IP and **syncs** the constructed inventory automatically (no manual **Sync** in the UI).

If `Demo-Multicloud` is still empty:

1. Confirm hosts exist under **Azure-Resources** and **AWS-Resources**.
2. Re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt` (post-tasks wire `input_inventories` and sync).
3. Run **Update - Multicloud inventory hosts** from **Templates**, or re-launch **WF - Demo setup**.
