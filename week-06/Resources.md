# Resources

A short list of references for this week's topics. Start with the official docs, then use the others when you want more context or a worked example.

---

## Terraform – Official

**Terraform Tutorials**
https://developer.hashicorp.com/terraform/tutorials

The main starting point. Work through the Azure track if you want step-by-step walkthroughs that match exactly what we did in class.

**Terraform on Azure – Get Started**
https://developer.hashicorp.com/terraform/tutorials/azure-get-started

The Azure-specific tutorial series from HashiCorp. Covers provider setup, building infrastructure, changing it, and destroying it.

**azurerm Provider Documentation**
https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs

The reference for every Azure resource type. When you need to know what arguments a resource takes — this is where to look.

**Terraform Language Documentation**
https://developer.hashicorp.com/terraform/language

Covers the full HCL language: variables, outputs, locals, modules, state, and more. Useful when you move beyond the basics.

---

## Azure – Official

**Terraform on Azure – Microsoft Docs**
https://learn.microsoft.com/en-us/azure/developer/terraform/

Microsoft's own documentation for using Terraform with Azure. Includes guides for common scenarios like VMs, networking, and storage accounts.

**Terraform Fundamentals on Microsoft Learn**
https://learn.microsoft.com/en-us/training/paths/terraform-fundamentals/

A free structured learning path from Microsoft covering variables, outputs, modules, and loops with Azure examples. Good if you want something more guided than the raw docs.

---

## GitHub Actions

**GitHub Actions – Quickstart**
https://docs.github.com/en/actions/writing-workflows/quickstart

Start here if you are new to GitHub Actions. Explains the basics of workflows, jobs, and steps.

**hashicorp/setup-terraform Action**
https://github.com/hashicorp/setup-terraform

The official GitHub Action used to install Terraform in a pipeline. The README shows exactly how to wire up init, plan, and apply steps.

**azure/login Action**
https://github.com/Azure/login

The official action for authenticating to Azure from a GitHub Actions workflow using a service principal or OIDC.

---

## Further Reading

**Terraform Modules – HashiCorp Docs**
https://developer.hashicorp.com/terraform/language/modules

Explains how modules work, how to call them, and how to pass inputs and read outputs. Directly relevant to Exercise 1.

**Store Terraform State in Azure Storage**
https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage

Microsoft's guide on setting up remote state with Azure Blob Storage — the same approach used in this lab.
