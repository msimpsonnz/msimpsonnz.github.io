---
layout: post
title: Hashicorp - Terraform and Packer
---

## Building clouds

I've been using specific tools like [CloudFormation](https://aws.amazon.com/cloudformation) and [ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview) for a number of years, so when I found out about [Teraform](https://www.terraform.io) from [HashiCorp](https://www.hashicorp.com/) I was keen to check it out.

I was involved in a project recently where Terraform was already being used and we wanted to run up some resources on Azure. This was a great opportunity to take a more in depth look at a larger scale deployment.

On the face of it Terraform has a number of resources to make it easy to get started, there are a number of providers for all the major cloud vendors and they are also working with projects like Kubernetes to package deployments. So standing up something in isolation was simple.

But what if you wanted to do this for a larger deployment, having everything in one single template doesn't feel like a great idea, but how do you stitch these things together and have things like dependencies? [Modules](https://www.terraform.io/docs/modules/index.html) are a great way to achieve this, I can have a something that is a standard template and reuse this throughout my environment deployment.

So I now had a simple deployment that would build the Azure Network Infrastructure as a module, I could then reuse this template for Test, Staging and Production environments. I definitely think this approach has it's merits when compared to the ARM equivalent of [Nested Templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-linked-templates).

I then added my [Azure Container Service](https://docs.microsoft.com/en-us/azure/container-service/kubernetes/container-service-kubernetes-walkthrough) provider and was able to deploy the K8s cluster on the custom VNET I created with the previous Terraform module, which took care of the provisioning order as it figured out the dependency.

### State of the nation

So I had a couple of modules and repeatable deployment, but as got deeper into this, I realised that the state engine was going to be an interesting concept and something that I needed to get right!

I found this out the hard way! I setup the [Azure backend State provider](https://www.terraform.io/docs/backends/types/azurerm.html) options, which allows you to store the state file in Azure Blob storage. Then I messed around with the blob naming scheme as I wasn't happy with is, this resulted in corrupting my state, which ended up having to import this again from the running Azure resources which was a pain.

Digging in on state, I found the general recommendation was to use a separate Terraform "repo" for each "service" I was creating, so I had a "VNET" service, "ACS" service and would soon scale out to "App Frontend" and "App Backend" services, all requiring their own state and individual configurations.

This was fine, until I found an open GitHub [issue](https://github.com/hashicorp/terraform/issues/516) around the state engine storing sensitive info, this was opened a few years ago and a little concerning that a good solution had not been found yet (as of Apr 18). A few people had suggested using private source control and some workarounds for secrets but the general guidance for Terraform is not to use source control for state, hence, why the state backend feature was there.

This was all good though, learning about the inner workings and finding out some of these things was what this was all about. I managed to get this working with ACS in a semi production view of how a good Terraform project should look, my repo is [here](https://github.com/msimpsonnz/tf-azure).

## Packer

As part of this project, we also need to deploy some dedicated VM to run a few different services like Mongo and Elastic. Wouldn't it be great if we could have pre-baked machines that could be deployed from a master image in the cloud. Enter [Packer](https://www.packer.io/intro/index.html), another HashiCorp open source project which does exactly that. The project already had some images using Packer built for a different cloud, but it was just a simple task to change the provider to target Azure.
Packer then does the hard work of creating a new VM, running the build commands and creating an [Azure VM Image](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/cli-ps-findimage) which can then be used in the VM deployment, there is a great walk through [here](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/build-image-with-packer).

## HashiCorp

Great to get some practical use of these tools and keen to explore the other offerings they have in [Vault](https://www.vaultproject.io/) and [Consul](https://www.consul.io/).