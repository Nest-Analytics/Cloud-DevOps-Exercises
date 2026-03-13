# Terraform – How It Works

This file explains what happens when you run Terraform commands and how the pieces of this repo fit together.

---

## Execution Flow

```
Write Terraform Code
        │
        ▼
terraform init
        │
        ▼
terraform plan
        │
        ▼
terraform apply
        │
        ▼
Infrastructure created in Azure
        │
        ▼
State file updated
```

---

## What Each Command Does

### `terraform init`

Downloads the required provider plugins and sets up the working directory. For this repo that means pulling down the Azure (`azurerm`) provider.

Run this once before anything else. You only need to run it again if you add a new provider or change the backend config.

### `terraform plan`

Reads your `.tf` files, checks the current state, and prints out exactly what it intends to do — without touching anything in Azure.

A typical output looks like:

```
Plan: 3 to add, 0 to change, 0 to destroy
```

Get into the habit of reading the plan before every apply. If you see an unexpected destroy, stop and find out why before continuing.

### `terraform apply`

Runs the plan and creates the infrastructure in Azure. Terraform will ask you to type `yes` to confirm before it does anything.

When it finishes, the state file is updated to reflect what now exists.

---

## State

Terraform keeps track of everything it has created in a state file (`terraform.tfstate`). This is how it knows what already exists and what needs to change on the next run.

In this repo the state is stored remotely in Azure Blob Storage so the pipeline and your local machine stay in sync. Do not edit or delete the state file manually.

---

## Repo Structure

```
├── main.tf                   # Root config — calls the VM module
├── variables.tf              # Input variables
├── outputs.tf                # Values printed after apply
├── backend.tf                # Remote state config
│
├── modules/
│   └── vm/
│       ├── main.tf           # VM, NIC, and Public IP resources
│       ├── variables.tf      # Module inputs
│       └── outputs.tf        # Module outputs (e.g. public IP address)
│
└── .github/
    └── workflows/
        └── terraform.yml     # GitHub Actions pipeline
```

### Root vs module

The root `main.tf` creates the Resource Group, Virtual Network, and Subnet. It then calls the `vm` module and passes in the values the module needs (like the subnet ID).

The `vm` module is responsible for the Public IP, Network Interface, and Virtual Machine. This separation keeps each piece focused and easy to test on its own.

---

## A Note on Variables

Values like the VM size, location, and resource group name are defined in `variables.tf` rather than hardcoded. This means you can change them in one place without hunting through every file.

When running locally you can override a variable on the command line:

```bash
terraform apply -var="location=westeurope"
```

Or create a `terraform.tfvars` file:

```hcl
location = "westeurope"
vm_size  = "Standard_B2s"
```

---

## Quick Reference

| Command | What it does |
|---|---|
| `terraform init` | Download providers, set up backend |
| `terraform plan` | Preview changes, nothing is created |
| `terraform apply` | Create or update infrastructure |
| `terraform destroy` | Remove everything Terraform manages |
| `terraform output` | Print output values after apply |
