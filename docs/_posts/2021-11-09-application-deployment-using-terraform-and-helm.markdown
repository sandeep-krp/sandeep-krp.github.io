---
layout: post
title:  "Simple steps to deploy applications on Kubernetes using terraform"
date:   2023-01-29 18:00:00 +0530
categories: tech ansible helm
---

### This article is for you if:
- You use terraform to deploy your cloud infrastructure but wondering how to use terraform to deploy applications to the managed kubernetes instance on your cloud
- Well, not just cloud, you want to understand how can you use terraform to install applications on Kubernetes

### Prerequisite
- You have already worked with terraform a little bit and have terraform installed
- You know how to install applications on Kubernetes with helm

### Basic example to install an application using helm
In this section we will take Jenkins as a sample application that we want to deploy using terraform
### Create provider.tf file
If you have worked a little with terraform you would already know that the providers are terraform's way to get the code from the internet to deploy certain stuff. For example, if you want to create AWS resources, you configure a `aws` provider. On the similar lines, if you want to install something using helm, you need a [helm provider](https://registry.terraform.io/providers/hashicorp/helm/latest/docs). Similar to other providers, the helm provider also needs a small bit of configuration to work. For starters, you can have the configuration as:
```
provider "helm" {
    kubernetes {
        config_path = "~/.kube/config"
    }
}
```
Note that this provider configuration should only be used for experiments and not in the production code. For production you need a configuration that is not dependent upon the machine from where the terraform will run. Everything should come from git itself. For example, if you are using AWS EKS as your target kubernetes server, the configuration can be:
```
provider "helm" {
    kubernetes {
        host                   = var.cluster_endpoint
        cluster_ca_certificate = base64decode(var.cluster_ca_cert)
        exec {
        api_version = "client.authentication.k8s.io/v1beta1"
        args        = ["eks", "get-token", "--cluster-name", var.cluster_name]
        command     = "aws"
        }
    }
}
```
In the above snippet, the `cluster_endpoint`,`cluster_ca_cert` and `cluster_name` variables can either come as an input to the module or can be referenced from a local resource (when you are creating the EKS from the same module)
For now, the first configuration will work for you.

### Create grafana.tf file
Let's create file called `grafana.tf` in your existing code or a completely new directly. Let's assume we are working on a fresh directly so that it is easy to understand. So we create a new directly anywhere on our machine and create a `grafana.tf` file. The name of the file could be anything as you would already know.\
So to start on a clean slate, create a new folder and start creating the following files.\
`grafana.tf`
```
resource "helm_release" "grafana" {
    name = "grafana"
    repository = "https://charts.bitnami.com/bitnami"
    chart = "grafana"
    version = "8.2.25"
    namespace = "grafana"
    create_namespace = true
    set_sensitive {
    name = "admin.password"
    value = var.default_admin_password
    }
}
```
The next file is `provider.tf`:
```
provider "helm" {
kubernetes {
        config_path = "~/.kube/config"
    }
}
```
And the last one is `variables.tf`:
```
variable "default_admin_password" {
    default = "admin"
}
```
Once you have created all the above files, move that folder and run:\
`terraform apply`
Type `yes` when it asks for the confirmation, and the Grafana helm chart would get deployed on the Kubernetes server pointed by the current context of `~/.kube/config`. If you would like to verify, you can run `kubectl port-forward svc/grafana -n grafana 3000:3000` and open `localhost:3000` on your browser to see Grafana UI.

If you are lazy to create all this code, just do:
```
git clone https://github.com/sandeep-krp/terraform-examples.git
cd terraform-examples/helm-basic
```
You will find all the code you need.
