---
title: "My Experience With Azure Key Vault and VM via Terraform"
date: 2024-12-29T20:30:21-07:00
tags: [Azure, Terraform]
---
Thanks to the [KubeCraft Community](https://github.com/mischavandenburg/kubecraft), I've recently had the opportunity to get my hands dirty with some Terraform and Azure.
<!--more-->
Since this community project was announced, I've been ecstatic to jump in to contribute.

## The What

Terraform is an infrastructure as code tool, meaning, we can deploy cloud infrastructure through configuration files.

One advantage of Terraform is that it manages multiple cloud platforms. Rather than manually configuring each cloud deployment on your chosen platform, you could have Terraform serve as a 'control panel' for all of your cloud providers.

Each platform you're deploying to provides its own set of resource definitions. For instance, compute instances or private networks can be single units of defined infrastructure in Azure.

Terraform allows you to sort of 'translate' these individual units between platforms, essentially standardizing your workflow. I find this portability between environments reminiscent of the [twelve factor app](https://12factor.net/). (Thank you, Zettelkasten!)

**note**: I have a [showcase](https://clumpspirit.com/posts/terraform-universal-panel/) on this.

Terraform also supports version control through a repository, and also allows you to deploy infrastructure with continuous deployment.

This means you can commit your changes to a repository, and it will automatically be released to your provider deployment, much like [Flux](https://fluxcd.io/) in a Kubernetes cluster.

Furthermore, I found that terraform is to infrastructure as Kubernetes is to virtual containers. They both orchestrate resources into the desired state defined by the admin. This is part of the reason I love working with these tools.

Contrarily, procedural programming languages, such as Java, require exact line-by-line input.

## The Process

Since the project is new, we had to build a base Terraform configuration from scratch.

Prior to starting, I knew we needed to structure our directory with these files.

### 1. main.tf

The heart and soul of the operation. Defines all necessary resources for a Key Vault and VM.

Besides containing all of the essential definitions for a an Azure VM and Key Vault, there are a few core concepts I was going for in each Terraform file:

- *Keep it minimal.*
    - The code reflects the ideology that you should start with something that works, then add as you need.

- *Separate access policies.*
    - There are two policies, one for the External Secrets Operator to be used with Kubernetes, and one for the admin of the Azure Key Vault deployment.

- *Keep access bare.*
    - The external secrets operator only needs a few permissions, don't give it more than it needs! All secrets are stored in the vault and the external secrets operator is just fetching them and creating secrets in your cluster for applications.

- *Try to keep things static.*
    - In order to reduce the need for input, I avoided the use of variables. This is because there is only one deployment environment so I didn't see the need for flexibility.

### 2. providers.tf

Integrates with remote API.

- *Use stable versions.*

    - To avoid any potential breaks, I avoided the use of the most recent provider versions.

### 3. locals.tf

Establishes some expressions to be used for easy & redundant reference for resources.

- *Limit it to volatile expressions.*

    - I wanted the locals file to serve as an ease-of-management pane and only put stuff that might be changed in production.

### 4. ssh.tf

Generates necessary keys for remote access from your local device.

This file is mostly copied from the Microsoft example, it just creates a random key name in our resource group from main.tf.

For these files, I often referenced [Microsoft's](https://learn.microsoft.com/en-us/azure/key-vault/keys/quick-create-terraform?tabs=azure-cli) and [HashiCorp's](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault)
example configurations. Moreover, I found that the best approach was to blend the best of both examples. One clear takeaway I had was that there are a plethora of solutions for the cloud, but you should apply what makes the most sense for your architecture.

For example, some solutions online will have a vault key alongside the vault; however, I'll avoid the unnecessary overhead and opt out of the vault key. To get started with an external secrets operator, you definitely don't need it.

Just make sure the simple stuff works.

## The Results

In the end, we achieved a working Azure environment containing a VM and Key Vault. All of the results are open-source and thus are shown in the [community repository](https://github.com/mischavandenburg/kubecraft/tree/main/terraform).
