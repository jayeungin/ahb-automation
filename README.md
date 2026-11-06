# Azure Hybrid Benefit for RHEL – automation

Ansible playbook that switches pay-as-you-go (PAYG) **RHEL** and **RHEL for SAP** Marketplace VMs in a
customer's Azure environment to Azure Hybrid Benefit (`licenseType: RHEL_BYOS`). After conversion the VMs are
billed against the customer's Red Hat Cloud Access subscriptions instead of Azure's PAYG RHEL rate.

The playbook runs a **dry run by default**. It changes customer billing, so nothing is modified unless you pass
`-e dry_run=false`.

## What it does

1. Checks that the Azure CLI is logged in and the `resource-graph` extension is installed.
2. Uses Azure Resource Graph to find every VM that:
   - was deployed from a Red Hat Marketplace image (publisher `RedHat`), for example the `RHEL`, `RHEL-HA`,
     `RHEL-SAP-HA` and `RHEL-SAP-APPS` offers
   - is not from a bring-your-own-subscription image (excluded by image offer and purchase plan)
   - is not already on `RHEL_BYOS`

   VMs in any power state are included, so stopped and deallocated VMs are converted too. Results are paged,
   so there's no limit on the number of VMs.
3. Sets `licenseType` to `RHEL_BYOS` on each of those VMs (skipped in dry run).
4. Prints a summary and fails the run if any VM couldn't be converted.

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
- Ansible with the `azure.azcollection` collection and its Python requirements:
  ```sh
  ansible-galaxy collection install azure.azcollection
  pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
  ```
- Azure CLI with the Resource Graph extension:
  ```sh
  az extension add --name resource-graph
  ```

**Azure permissions** for the identity you log in with:
- Reader on the scope being searched (the tenant root management group to cover everything)
- `Microsoft.Compute/virtualMachines/write` on the target VMs (for example, Virtual Machine Contributor)

## Usage

Log in first. The Resource Graph queries and the license changes both use this login.

```sh
az login --tenant <customer-tenant-id>
```

Dry run (default). Lists what would change:

```sh
ansible-playbook enable_ahb.yml
```

Dry run across every subscription under a management group:

```sh
ansible-playbook enable_ahb.yml -e management_group_id=<root-management-group-id>
```

Apply the changes:

```sh
ansible-playbook enable_ahb.yml -e management_group_id=<root-management-group-id> -e dry_run=false
```

If you don't set `management_group_id`, the playbook searches every subscription the logged-in identity can see.

## Variables

| Variable | Default | Description |
|---|---|---|
| `dry_run` | `true` | When `true`, discover and report only. Set to `false` to apply changes. |
| `management_group_id` | `""` | Management group to search. Empty means all subscriptions the identity can see. |
| `azure_auth_source` | `cli` | How `azure.azcollection` modules authenticate. `cli` reuses your `az login`. |
| `target_license_type` | `RHEL_BYOS` | License type applied to the target VMs. |

## Output

The run ends with a summary like this:

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
2. Convert a small test set first, including at least one deallocated VM.
3. Run the full conversion.
4. Handle the report-only VMs and scale sets separately.

## Files

| Path | Purpose |
|---|---|
| `enable_ahb.yml` | Main playbook |
| `tasks/arg_query.yml` | Runs a Resource Graph query and fetches every page of results |
| `enable_ahb.yml.orig` | Original version of the playbook, kept for reference |

## Reverting

To return a VM to PAYG billing, set its license type back to `None`:

```sh
az vm update -g <resource-group> -n <vm-name> --license-type None
```
