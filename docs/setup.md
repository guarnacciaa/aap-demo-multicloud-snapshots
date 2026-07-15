# Environment setup

## Deployment mode: lab/dev vs customer/PoC

Set `demo_manage_infrastructure` in `group_vars/all/demo_variables.yml` **before** running `aap_config.yml`:


| `demo_manage_infrastructure` | Scenario                                                                             | What CasC creates                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `true` (default)             | Lab, dev, or self-running the full demo; no pre-existing Azure VM / AWS EC2 instance | Provisioning (`Provision - *`), inventory sync (`Update - Multicloud inventory hosts`), teardown (`Deprovision - *`) job templates, `WF - Demo setup`, `WF - Demo teardown`, plus all snapshot objects |
| `false`                      | Deploying at a customer site where the VM/instance and its networking already exist  | Only the snapshot job templates (`Snapshot - Connectivity check (dry run)`, `Snapshot - Preview (dry run)`, `Snapshot - Azure/AWS by hostname`, `Snapshot - Verify`, `Snapshot - Cleanup preview (dry run)`, `Snapshot - Cleanup (optional)`, `Snapshot - Inventory report`) and `WF - Multicloud snapshot and retention`                                |


The flag is read in `playbooks/aap_config.yml`: when `true`, the objects defined in `group_vars/all/job_templates_infra.yml` and `group_vars/all/workflow_templates_infra.yml` are merged into the lists the `infra.aap_configuration` dispatch role applies; when `false`, those two files are still loaded (Ansible auto-loads every file under `group_vars/all/`) but never merged in, so their objects are never created in AAP.

### Which variables to set in each mode

`group_vars/all/demo_variables.yml.example` **and** `vault.yml.example` group every variable/secret under the same banner:

- `[ALWAYS REQUIRED]` — needed regardless of mode: AAP connection, object names, Git repo, snapshot policy, credential names, the target VM/instance identity (`azure_resource_group`, `azure_vm_hostname`, `aws_ec2_hostname`, `aws_region`), and the Azure/AWS credential secrets (`vault_azure_client_id`, `vault_azure_client_secret`, `vault_aws_access_key`, `vault_aws_secret_key`) — the snapshot job templates authenticate against Azure/AWS in both modes. The two Azure secrets are further conditional on a second, independent axis: `azure_auth_mode` (see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity)) — only required when it is `service_principal` (the default), unused when it is `msi`.
- `[LAB/DEV ONLY]` — consumed exclusively by `Provision - *` / `Deprovision - *`: VM size, admin user/password, image, VNet/VPC/subnet/NSG/SG names and CIDRs, AMI ID, key pair in `demo_variables.yml.example`, plus `vault_azure_vm_admin_password` in `vault.yml.example`. Leave these at their example defaults in customer/PoC mode; they are never read.

Both `playbooks/aap_config.yml` (pre-task assertions) and `playbooks/verify.yml` enforce this split: the `[LAB/DEV ONLY]` checks only run `when: demo_manage_infrastructure | bool`, so a customer/PoC deployment fails fast only on the variables it actually needs, never on unrelated provisioning variables.

When `demo_manage_infrastructure: false`:

- Set `azure_vm_hostname` / `azure_resource_group` to the customer's existing VM name and resource group.
- Set `aws_ec2_hostname` / `aws_region` to the customer's existing EC2 instance name (must match its `Name` tag, since the snapshot playbook filters on `tag:Name`) and region.
- Request Azure/AWS credentials scoped to **read + snapshot only** (see [Reduced credential scope for customer/PoC mode](#reduced-credential-scope-for-customer-poc-mode) below) — the full provisioning permissions in [Azure prerequisites](#azure-prerequisites) and [AWS prerequisites](#aws-prerequisites) are not needed since no provisioning/deprovisioning job template exists to use them.
- You do not need `vault_azure_vm_admin_password`; it is `[LAB/DEV ONLY]` in `vault.yml.example` and only consumed by `provision_azure_vm.yml`.
- Switching modes later: change the flag and re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt`. Going from `true` to `false` does **not** remove the infra job/workflow templates already created in AAP (dispatch only reconciles objects it is told about); delete them explicitly with `ansible-playbook playbooks/aap_cleanup.yml -e demo_cleanup_confirm=true --vault-id @prompt` and re-apply, or delete them manually from the Controller UI.



### Reduced credential scope for customer/PoC mode

When `demo_manage_infrastructure: false`, request credentials limited to:

- **Azure Service Principal**: `Microsoft.Compute/virtualMachines/read`, `Microsoft.Compute/disks/read`, `Microsoft.Compute/snapshots/read`, `Microsoft.Compute/snapshots/write`, `Microsoft.Compute/snapshots/delete` on the resource group containing the existing VM.
- **AWS IAM user or role**: `ec2:DescribeInstances`, `ec2:CreateSnapshot`, `ec2:DescribeSnapshots`, `ec2:DeleteSnapshot`, `ec2:CreateTags`.

Do not request the VPC/VNet/subnet/NSG/security group/route table/internet gateway create-delete permissions listed below; they are only used by the provisioning and teardown playbooks, which are not deployed in this mode.

## Azure authentication mode: Service Principal vs Managed Identity

`azure_auth_mode` in `group_vars/all/demo_variables.yml` (default `service_principal`) controls how every `azure.azcollection` task (snapshot, provisioning, and teardown playbooks) authenticates to Azure. This is independent of `demo_manage_infrastructure` above — it applies to the snapshot playbooks in every deployment mode.


| `azure_auth_mode`             | How it authenticates                                                                                                                                                                                             | Requirements                                                                                                                                                                                                                                                                                                                                                                                                             |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `service_principal` (default) | AAP's "Microsoft Azure Resource Manager" credential injects `AZURE_CLIENT_ID`/`AZURE_SECRET`/`AZURE_TENANT`/`AZURE_SUBSCRIPTION_ID`; playbooks omit `auth_source` so modules use their default `auto` resolution | `vault_azure_client_id` and `vault_azure_client_secret` in `vault.yml`. Works regardless of where AAP is hosted.                                                                                                                                                                                                                                                                                                         |
| `msi`                         | Every `azure_rm_*` task passes `auth_source: msi` explicitly; no client secret is stored in AAP at all                                                                                                           | The AAP execution node or execution environment container that actually runs the job must itself be an Azure resource (VM, VMSS, AKS node, etc.) with a system- or user-assigned Managed Identity enabled, and that identity must hold the same RBAC role documented in [Azure prerequisites](#azure-prerequisites) / [Reduced credential scope for customer/PoC mode](#reduced-credential-scope-for-customer-poc-mode). |


Notes:

- AAP's built-in Azure credential type has **no Managed Identity input field** — it only supports Service Principal or Active Directory user/password. `auth_source: msi` is a capability of the `azure.azcollection` modules themselves, not something the Controller credential injects. The Azure credential is still created in `msi` mode (with only `azure_subscription_id` populated) so job templates keep a stable credential association, but `client`/`secret`/`tenant` are omitted entirely — no secret is ever stored.
- Choose `msi` only when you know AAP's execution nodes run on Azure infrastructure with an identity attached. In every other topology (on-prem, another cloud, OpenShift not on Azure), `service_principal` is the only option that works.
- `playbooks/aap_config.yml` and `playbooks/verify.yml` validate `azure_auth_mode` and only require the Service Principal vault secrets when it is set to `service_principal` (the default).
- Switching modes: change `azure_auth_mode` and re-run `ansible-playbook playbooks/aap_config.yml --vault-id @prompt` to update the Azure credential's stored inputs.
- **`azure_auth_mode` must be threaded through each job template's `extra_vars`** (see `group_vars/all/job_templates.yml` / `job_templates_infra.yml`). AAP runs `playbooks/demo/*.yml` against its own generated inventory, not this repository's `inventory.yml`, so `group_vars/all/demo_variables.yml` is **not** auto-loaded for those nested playbooks the way it is for `playbooks/aap_config.yml` and `playbooks/verify.yml`. If a job template is missing `azure_auth_mode` in its `extra_vars`, every `auth_source` Jinja expression in that playbook silently falls back to the `service_principal` branch — even with `azure_auth_mode: msi` set correctly in `demo_variables.yml` — and Azure auth fails with a "Failed to get credentials" error mentioning environment variables, a `~/.azure/credentials` profile, and `az login`, because the MSI credential (subscription only, no client/secret) does not satisfy `auto` resolution either. Re-run `aap_config.yml` after editing `extra_vars` so the stored job template picks up the change.



## Requirements


| Component                           | Version / detail                                                                                            |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Red Hat Ansible Automation Platform | 2.6 or later                                                                                                |
| Ansible Core                        | 2.16+ (bundled with AAP)                                                                                    |
| Azure subscription                  | With permissions to create resource groups, VNets, NSGs, public IPs, NICs, and VMs                          |
| AWS account                         | With permissions to create VPCs, subnets, IGWs, route tables, security groups, key pairs, and EC2 instances |
| Network                             | AAP must reach both Azure and AWS API endpoints                                                             |




## Trust the platform CA (growth installs)

Do this **before** any other operation against this AAP instance: `podman login`/`push`, `ansible-galaxy collection install`, or `ansible-playbook playbooks/aap_config.yml`. On container **growth** (all-in-one) deployments, Private Automation Hub and Controller both serve a self-signed certificate by default. Left untrusted, this causes `x509: certificate signed by unknown authority` errors at multiple, unrelated points later on: `podman login`/`push` to the Hub registry, Controller pulling the custom execution environment image, and sometimes `ansible-galaxy`/API calls from the bastion.

Install the AAP install CA on any node that will run `podman` or `ansible` commands against this platform (run as root):

```bash
# Default path for the containerized installer; adjust if your install dir differs.
sudo cp ~/aap/tls/ca.cert /etc/pki/ca-trust/source/anchors/aap-platform-ca.crt
sudo update-ca-trust extract
```

This is a one-time step per node; the containerized installer does not always copy the CA into the system trust store automatically. If you skip it, use `--tls-verify=false` for `podman login`/`push` and rely on the **Verify SSL**-disabled Container Registry credential (created by CasC) for Controller's own EE pulls — see [Custom Execution Environment (optional)](#custom-execution-environment-optional) below — but note that credential alone does not always cover EE pulls on growth installs (see the troubleshooting table).

## Azure prerequisites

- **Service Principal** (default, `azure_auth_mode: service_principal`) **or Managed Identity** (`azure_auth_mode: msi`, see [Azure authentication mode](#azure-authentication-mode-service-principal-vs-managed-identity)) with the following permissions (Contributor role on the subscription or resource group scope is sufficient):
  - Resource group: create, read, delete
  - Virtual network and subnet: create, read, delete
  - Network security group: create, read, delete
  - Public IP address: create, read, delete
  - Network interface: create, read, delete
  - Virtual machine: create, read, delete
  - Managed disk and snapshot: create, read, delete
- Service Principal mode: note the **subscription ID**, **tenant ID**, **client ID**, and **client secret**. Managed Identity mode: note only the **subscription ID**; grant the RBAC role above to the identity itself instead of a Service Principal.



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


| Vault variable                  | Purpose                                           | Mode                                                                                                        |
| ------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `vault_controller_password`     | AAP admin password                                | `[ALWAYS REQUIRED]`                                                                                         |
| `vault_azure_client_id`         | Azure Service Principal client (application) ID   | `[ALWAYS REQUIRED]` when `azure_auth_mode: service_principal` (default); unused with `azure_auth_mode: msi` |
| `vault_azure_client_secret`     | Azure Service Principal client secret             | `[ALWAYS REQUIRED]` when `azure_auth_mode: service_principal` (default); unused with `azure_auth_mode: msi` |
| `vault_azure_vm_admin_password` | Azure VM admin password (set during provisioning) | `[LAB/DEV ONLY]`                                                                                            |
| `vault_aws_access_key`          | AWS IAM access key ID                             | `[ALWAYS REQUIRED]`                                                                                         |
| `vault_aws_secret_key`          | AWS IAM secret access key                         | `[ALWAYS REQUIRED]`                                                                                         |




## Custom Execution Environment (optional)

If your platform EE lacks cloud SDKs, build a custom image from `context/execution-environment.yml`. The definition uses the AAP 2.7 `ee-supported-rhel9` base image from `registry.redhat.io` (not the legacy `quay.io/ansible/ansible-runner` default).

**Prerequisites**

1. Log in to the Red Hat container registry and pull the base image:
  ```bash
   podman login registry.redhat.io
   podman pull registry.redhat.io/ansible-automation-platform-27/ee-supported-rhel9:latest
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

If job templates fail to pull the image:

1. In **Automation Execution** > **Infrastructure** > **Execution Environments** > `ee-multicloud-snapshots`, confirm **Pull credential** is **PAH Container Registry** and open that credential: **Verify SSL** must be **off**.
2. Confirm `hub_registry_host` matches the registry host in `demo_execution_environment_image` exactly (same IP/hostname and port; if you pushed to `<aap_host>:8444`, both must include `:8444`).
3. Re-apply CasC (post-task wires the credential explicitly):
  ```bash
   ansible-playbook playbooks/aap_config.yml --tags credentials,execution_environments --vault-id @prompt
  ```

For local `podman pull` tests, use `--tls-verify=false` (Controller job pulls use the credential instead). If the pull still fails with `x509: certificate signed by unknown authority` after confirming the credential, you likely skipped [Trust the platform CA (growth installs)](#trust-the-platform-ca-growth-installs) above — that system-level trust step is needed on growth installs in addition to the **Verify SSL**-disabled credential.

**Troubleshooting**


| Symptom                                                                                         | Likely cause                                                                                                                                 | Fix                                                                                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `HTTP Code: 401, Message: Unauthorized` during `ansible-galaxy collection install` in the build | Missing or placeholder token in `../ansible.cfg`, or `ansible.cfg` not created at artifact root                                              | Create `ansible.cfg` from the example and set a valid Hub token; confirm with the local `ansible-galaxy` command above                                                                                                       |
| `explicit_requirement_infra.aap_configuration ... 401`                                          | Old `context/requirements.yml` with per-collection `source:` URLs or validated collections in the EE                                         | Use the current `context/requirements.yml` (certified cloud collections only; no `source:` keys)                                                                                                                             |
| Warning about `quay.io/ansible/ansible-runner`                                                  | Missing `images.base_image` in an old definition file                                                                                        | Use the current `context/execution-environment.yml` with `ee-minimal-rhel9`                                                                                                                                                  |
| `/usr/bin/python3: No module named pip` in the builder stage                                    | Explicit `python:` / `python3-pip` entries forced a UBI Python stack into the builder                                                        | Use the current galaxy-only `context/execution-environment.yml`; collections supply pip deps                                                                                                                                 |
| `ModuleNotFoundError: No module named 'yaml'` in `introspect.py`                                | Same root cause: microdnf `python3-pip` in the builder stage conflicts with `ee-minimal-rhel9`                                               | Remove custom `prepend_builder` / system pip packages; rebuild from a clean `context/_build` directory                                                                                                                       |
| `image not known` on `podman push`                                                              | Image not tagged for the Hub registry hostname                                                                                               | Run `podman tag localhost/ee-multicloud-snapshots:latest <aap_host>/ee-multicloud-snapshots:latest` before push                                                                                                              |
| `x509: certificate signed by unknown authority` on `podman login`                               | Self-signed or internal CA certificate on the Hub registry                                                                                   | Use `podman login --tls-verify=false` and `podman push --tls-verify=false`; disable **Verify SSL** on the Container Registry credential in AAP                                                                               |
| `401 Unauthorized` on `podman login` or push                                                    | Wrong AAP credentials or insufficient Hub permissions                                                                                        | Use a user with permission to create containers on Hub (for example the gateway admin); authenticate through the Platform Gateway                                                                                            |
| Connection refused on push                                                                      | Wrong registry host or port                                                                                                                  | Confirm `<aap_host>` matches `aap_hostname`; retry with port `8444` as shown above                                                                                                                                           |
| `couldn't resolve module/action 'azure.azcollection.*'` at job runtime                          | Job uses the placeholder EE (`quay.io/ansible/ansible-runner`) instead of the custom image                                                   | Set `demo_execution_environment_image` in `demo_variables.yml`, re-run `aap_config.yml`, and confirm the job template points to `ee-multicloud-snapshots`                                                                    |
| `x509: certificate signed by unknown authority` when Controller pulls the EE                    | Hub registry uses a self-signed certificate; EE missing registry credential and/or host OS does not trust the platform CA                    | Link **PAH Container Registry** with **Verify SSL** off to the EE; if pull still fails on growth installs, see [Trust the platform CA (growth installs)](#trust-the-platform-ca-growth-installs) |
| `unable to copy from source docker://<host>/ee-...` with x509 error                             | Same as above, or `hub_registry_host` does not match the registry in `demo_execution_environment_image` (including `:8444` if used for push) | Align image host and credential; re-run CasC; see [Trust the platform CA (growth installs)](#trust-the-platform-ca-growth-installs)                                                                                                                      |
| `Demo-Multicloud` inventory shows no hosts                                                      | Constructed inventory missing `input_inventories`, or hosts only in child inventories                                                        | Re-run `aap_config.yml`; confirm hosts under **Azure-Resources** / **AWS-Resources**; run **Update - Multicloud inventory hosts** after setup workflow                                                                       |
| Sync task fails with HTTP 404 on `/api/v2/`                                                     | Legacy Controller API path on Platform Gateway                                                                                               | Use `ansible.controller` modules only; gateway path is `/api/controller/v2/` (AAP 2.5+)                                                                                                                                      |
| Sync task: inventory source not found                                                           | Wrong source name (`Demo-Multicloud` vs auto-created name)                                                                                   | Source is `Auto-created source for: Demo-Multicloud`; sync task resolves it via API lookup                                                                                                                                   |




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

