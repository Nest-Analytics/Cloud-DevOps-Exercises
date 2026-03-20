# Resources — Docker, Azure Container Apps, Terraform

---

## Conceptual Reading

**Why containers?**
[https://www.docker.com/resources/what-container/](https://www.docker.com/resources/what-container/) — Docker's own plain-English explanation of what a container is and why it exists.

**Containers vs virtual machines**
[https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/containers-vs-vm](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/containers-vs-vm) — short comparison of the two approaches and when each makes sense.

**How container registries work**
[https://www.redhat.com/en/topics/cloud-native-apps/what-is-a-container-registry](https://www.redhat.com/en/topics/cloud-native-apps/what-is-a-container-registry) — explains what a registry is, how images are stored and tagged, and the push/pull flow.

**The Twelve-Factor App — Processes and Port Binding**
[https://12factor.net/processes](https://12factor.net/processes) — the thinking behind why apps should be stateless and self-contained, which is exactly what containerisation enforces.

---

## Official Docs

### Docker

**Dockerfile reference**
[https://docs.docker.com/reference/dockerfile/](https://docs.docker.com/reference/dockerfile/) — every instruction explained. Bookmark this; you will come back to it.

**docker build**
[https://docs.docker.com/reference/cli/docker/buildx/build/](https://docs.docker.com/reference/cli/docker/buildx/build/) — flags and options for the build command.

**docker run**
[https://docs.docker.com/reference/cli/docker/container/run/](https://docs.docker.com/reference/cli/docker/container/run/) — all the flags for running containers, including port mapping and detached mode.

**.dockerignore**
[https://docs.docker.com/build/concepts/context/#dockerignore-files](https://docs.docker.com/build/concepts/context/#dockerignore-files) — what it does and how to write it correctly.

---

### Azure

**Azure Container Apps overview**
[https://learn.microsoft.com/en-us/azure/container-apps/overview](https://learn.microsoft.com/en-us/azure/container-apps/overview) — what the service is and when to use it.

**Deploy a container app (Azure CLI)**
[https://learn.microsoft.com/en-us/azure/container-apps/get-started](https://learn.microsoft.com/en-us/azure/container-apps/get-started) — the quickstart we followed in class, useful for reference.

**Ingress in Azure Container Apps**
[https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview](https://learn.microsoft.com/en-us/azure/container-apps/ingress-overview) — explains external vs internal ingress and how traffic reaches your app.

---

### Terraform

**azurerm_container_app resource**
[https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app) — the full argument reference for the resource you wrote in Exercise 2.

**azurerm_container_app_environment resource**
[https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app_environment](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app_environment) — the environment resource that sits beneath your app.

**Terraform input variables**
[https://developer.hashicorp.com/terraform/language/values/variables](https://developer.hashicorp.com/terraform/language/values/variables) — how variables work, including passing them via `-var` flags in the CLI and pipeline.
