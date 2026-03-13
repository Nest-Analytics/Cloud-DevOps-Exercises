
# Terraform Lab

This lab covers two things: writing a Terraform module to build a VM, and running Terraform through a GitHub Actions pipeline instead of your local machine.

---

## Exercise 1 – Create a VM Using a Terraform Module

The root configuration already sets up a Resource Group, Virtual Network, and Subnet. Your job is to complete the VM module so Terraform can also create:

- A Public IP
- A Network Interface
- A Linux Virtual Machine

### Steps

**1. Clone the repository**

```bash
git clone <repo-url>
cd <repo-folder>
```

**2. Open the module folder**

```
modules/vm/main.tf
```

This is where you will add the three missing resources. The root `main.tf` already calls the module — you just need to fill it in.

**3. Initialise Terraform**

```bash
terraform init
```

**4. Check what Terraform plans to create**

```bash
terraform plan
```

Read the output. Every resource listed with a `+` will be created. Make sure the VM, NIC, and Public IP are all in the list before you continue.

**5. Apply**

```bash
terraform apply
```

Type `yes` when prompted.

**6. Confirm in the portal**

Go to [portal.azure.com](https://portal.azure.com), open the resource group, and check that the VM appears.

### What you should see after apply

- `azurerm_public_ip` created
- `azurerm_network_interface` created
- `azurerm_linux_virtual_machine` created
- The Resource Group, VNet, and Subnet were already there from the root config

---

## Exercise 2 – Run Terraform from GitHub Actions

Rather than running Terraform on your machine, this exercise wires it into a CI/CD pipeline. Every push triggers the workflow, which runs init, plan, and apply for you.

### Steps

**1. Open the workflow file**

```
.github/workflows/terraform.yml
```

**2. Complete the missing steps**

The file has placeholders where the following steps need to be added:

- Checkout code
- Set up Terraform
- Log in to Azure
- `terraform init`
- `terraform plan`
- `terraform apply`

**3. Add the required secrets to your repository**

Go to your repo on GitHub:

```
Settings → Secrets and variables → Actions → New repository secret
```

Add these three secrets:

| Secret name | Where to find the value |
|---|---|
| `AZURE_CLIENT_ID` | Your service principal app ID |
| `AZURE_TENANT_ID` | Your Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Your subscription ID |

These let GitHub Actions authenticate to Azure without storing passwords in the code.

**4. Push your changes**

```bash
git add .
git commit -m "complete terraform workflow"
git push
```

**5. Watch the pipeline run**

Open the **Actions** tab in your GitHub repository and watch the workflow run step by step.

---

## Submission

Include the following in your submission:

1. Link to your GitHub repository
2. Screenshot of `terraform plan` output (local or from the Actions log)
3. Screenshot of `terraform apply` output
4. Screenshot of the VM in the Azure portal
5. Screenshot of a completed GitHub Actions workflow run

---

## Troubleshooting

**`terraform init` fails** — check that you are inside the correct folder and that your Azure CLI is logged in (`az login`).

**`Error: A resource with the ID already exists`** — run `terraform destroy` first, then apply again.

**GitHub Actions fails on the Azure login step** — double-check that all three secrets are set and spelled correctly (they are case-sensitive).

**VM does not appear in the portal** — make sure you are looking at the correct subscription and region in the portal top-right dropdown.
