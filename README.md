# Azure Hybrid Benefit for RHEL – automation

Ansible collection (`jayeungin.azure_ahb`) that switches pay-as-you-go (PAYG) **RHEL** and **RHEL for SAP**
Marketplace VMs in a customer's Azure environment to Azure Hybrid Benefit (`licenseType: RHEL_BYOS`). After conversion the VMs are
billed against the customer's Red Hat Cloud Access subscriptions instead of Azure's PAYG RHEL rate.

The playbook runs a **dry run by default**. It changes customer billing, so nothing is modified unless you pass
`-e enable_ahb_dry_run=false`.

## What it does

1. Uses the Azure Resource Graph REST API to find every VM that:
   - was deployed from a Red Hat Marketplace image (publisher `RedHat`), for example the `RHEL`, `RHEL-HA`,
     `RHEL-SAP-HA` and `RHEL-SAP-APPS` offers
   - is not from a bring-your-own-subscription image (excluded by image offer and purchase plan)
   - is not already on `RHEL_BYOS`

   VMs in any power state are included, so stopped and deallocated VMs are converted too. Results are paged,
   so there's no limit on the number of VMs.
2. Sets `licenseType` to `RHEL_BYOS` on each of those VMs (skipped in dry run).
3. Prints a summary and fails the run if any VM couldn't be converted.

It also produces two **report-only** lists, which it never modifies:

| List | Why it isn't converted automatically |
|---|---|
| RHEL VMs from custom or gallery images | They weren't deployed straight from a Red Hat Marketplace image, so how they're billed needs a manual check. |
| Uniform scale sets built from Red Hat images | The license type is set on the scale set model, and existing instances must be upgraded to pick it up. |

The AHB extension (`AHBForRHEL`) is not installed. It's only needed for the opposite conversion (BYOS to PAYG).

## Prerequisites

**Customer / licensing**
- The customer is enrolled in [Red Hat Cloud Access](https://access.redhat.com/public-cloud) with enough
  subscriptions for the VMs being converted. If RHEL for SAP VMs are in scope, the enrollment must include
  RHEL for SAP Solutions subscriptions too.

**Control node**

This is a CLI tool: run it with `ansible-playbook`, not `ansible-navigator`. By default it authenticates
with your `az login` session, which isn't visible from inside an execution environment container. (Red Hat
also doesn't ship an execution environment that includes `azure.azcollection`.)

`azure.azcollection` pulls in a lot of Python dependencies that often conflict with packages already on
the system, so install everything into a dedicated virtual environment:

```sh
python3 -m venv ~/venvs/azure-ahb
source ~/venvs/azure-ahb/bin/activate
pip install --upgrade pip
pip install ansible-core

# Installs this collection and its azure.azcollection dependency
ansible-galaxy collection install git+https://github.com/jayeungin/ahb-automation.git

pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

You also need the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) to run `az login`.
The `resource-graph` CLI extension isn't needed.

**Azure permissions** for the identity you log in with:
- Reader on the scope being searched (the tenant root management group to cover everything)
- `Microsoft.Compute/virtualMachines/write` on the target VMs (for example, Virtual Machine Contributor)

## Usage

Activate the venv and log in first. The Resource Graph queries and the license changes both use this login.

```sh
source ~/venvs/azure-ahb/bin/activate
az login --tenant <customer-tenant-id>
```

Dry run (default). Lists what would change:

```sh
ansible-playbook jayeungin.azure_ahb.enable_ahb
```

Dry run across every subscription under a management group:

```sh
ansible-playbook jayeungin.azure_ahb.enable_ahb -e enable_ahb_management_group_id=<root-management-group-id>
```

Apply the changes:

```sh
ansible-playbook jayeungin.azure_ahb.enable_ahb \
  -e enable_ahb_management_group_id=<root-management-group-id> \
  -e enable_ahb_dry_run=false
```

Running with `--check` is treated as a dry run too: discovery still runs, but no VM is modified, even with
`enable_ahb_dry_run=false`.

If you don't set `enable_ahb_management_group_id`, the playbook searches every subscription the logged-in
identity can see.

To use the role from your own playbook instead:

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: jayeungin.azure_ahb.enable_ahb
      vars:
        enable_ahb_dry_run: true
```

### Service principal instead of `az login`

Set `enable_ahb_auth_source=env` and export the standard `azure.azcollection` variables
(`AZURE_CLIENT_ID`, `AZURE_SECRET`, `AZURE_TENANT`, `AZURE_SUBSCRIPTION_ID`). The service principal needs the
same Azure permissions listed above. Nothing else changes.

## Variables

| Variable | Default | Description |
|---|---|---|
| `enable_ahb_dry_run` | `true` | When `true`, discover and report only. Set to `false` to apply changes. |
| `enable_ahb_management_group_id` | `""` | Management group to search. Empty means all subscriptions the identity can see. |
| `enable_ahb_auth_source` | `cli` | How `azure.azcollection` authenticates. `cli` reuses your `az login`; `env` uses environment variables. |
| `enable_ahb_target_license_type` | `RHEL_BYOS` | License type applied to the target VMs. |
| `enable_ahb_arg_page_size` | `1000` | Rows per Resource Graph page (service maximum is 1000). |

## Output

The run ends with a summary. A dry run looks like this:

```
Dry run - would convert to RHEL_BYOS: 42
Failed: []
Needs manual review (RHEL from custom/gallery image): ['rhel-golden-01']
Scale sets not modified: ['sap-web-vmss']
```

After applying changes, the first line reports what was actually converted:

```
Converted to RHEL_BYOS: 41 of 42
Failed: ['rhel-app-07']
Needs manual review (RHEL from custom/gallery image): ['rhel-golden-01']
Scale sets not modified: ['sap-web-vmss']
```

If any VM fails, the play exits non-zero after the summary. Errors for each VM appear in the output of the
`Update VM license type` task. The playbook is safe to re-run: VMs already on `RHEL_BYOS` are skipped.

## Recommended rollout

1. Run the dry run against the customer tenant and check the VM list against what you expect.
2. Convert a small test set first, including at least one deallocated VM. The playbook has no per-VM filter,
   so do this by pointing `enable_ahb_management_group_id` at a management group that contains only a test
   subscription, or by logging in as an identity that can only see the test subscription.
3. Run the full conversion.
4. Handle the report-only VMs and scale sets separately.

## Layout

| Path | Purpose |
|---|---|
| `galaxy.yml` | Collection metadata |
| `playbooks/enable_ahb.yml` | Playbook, run as `jayeungin.azure_ahb.enable_ahb` |
| `roles/enable_ahb/` | The role that does the work |
| `roles/enable_ahb/tasks/arg_query.yml` | Runs a Resource Graph query through the REST API and fetches every page |
| `roles/enable_ahb/vars/main.yml` | Resource Graph queries and API versions |

To test local changes, install from the working tree: `ansible-galaxy collection install . --force`.

## Reverting

To return a VM to PAYG billing, set its license type back to `None`:

```sh
az vm update -g <resource-group> -n <vm-name> --license-type None
```
