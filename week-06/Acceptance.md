# Acceptance Criteria

Submit the following to confirm you have completed the lab.

---

## 1. Repository

Provide the link to your GitHub repository.

The repository must contain:

```
main.tf
variables.tf
outputs.tf
modules/vm/
.github/workflows/terraform-deploy.yml
```

---

## 2. Terraform Commands

Provide a screenshot of each command running successfully.

- `terraform init`
- `terraform plan`
- `terraform apply`

---

## 3. Azure Portal

Provide screenshots showing the following resources exist in your Azure Portal:

- Resource Group
- Virtual Network
- Subnet
- Public IP
- Network Interface
- Linux Virtual Machine

---

## 4. GitHub Actions

Provide a screenshot of a completed pipeline run from the Actions tab.

The pipeline must include these steps:

- Checkout repository
- Setup Terraform
- Login to Azure
- Terraform Init
- Terraform Plan
- Terraform Apply

---

## 5. Short Questions

Answer each of the following in a few sentences:

1. What is a Terraform module?
2. Why are modules used in Terraform projects?
3. What does `terraform plan` do?
4. What does `terraform apply` do?
5. Why would a team run Terraform from a CI/CD pipeline rather than locally?

---

## Acceptance Criteria

Your submission is complete when:

- [ ] Terraform ran successfully (`init`, `plan`, `apply`)
- [ ] The VM was created using the module
- [ ] All six Azure resources are visible in the portal
- [ ] The GitHub Actions pipeline completed without errors
- [ ] All five questions are answered

---

## Important

- Do not commit secrets or SSH keys to the repository
- Use GitHub Secrets for all sensitive values (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`)
