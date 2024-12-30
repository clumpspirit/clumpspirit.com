---
title: "A brief demo of Terraform"
date: 2024-12-29T20:28:09-07:00
tags: [Terraform, Docker, Demo]
---

Here's a showcase Terraform's magical translative powers.
<!--more-->

Start with this basic configuration in a file called ```main.tf```.

```
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}
```

Next, initialize Terraform.

`terraform init`

Great! Terraform should tell us that it's ready to go now. If we do `terraform apply`, it will spawn 0 resources, naturally. We haven't defined them.

You may have noticed that we're using Docker as our provider through the provider plugin.

Add this configuration below the one we made:

```
provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx"
  keep_locally = false
}
```

Here, we're telling Terraform to use the Docker provider, and pull the nginx image.

Here's some magic, when you run `terraform plan` again, it will show this:

```
Terraform used the selected providers to generate the following
execution plan. Resource actions are indicated with the following
symbols:
  + create
```

Amazing! It recognized the create command that Docker would utilize for creating a container. We did not mention the create command anywhere in our configuration, Terraform was able to figure out through the plugin!
