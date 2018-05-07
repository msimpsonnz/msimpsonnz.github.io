---
layout: post
title: Azure Container Service Engine (aks ACS-Engine)
---

Just a quick update this week. I have been working on a project where we are going to migrate Kubernetes to Azure, we really wanted to offload the management using [AKS](https://docs.microsoft.com/en-us/azure/aks/), which is Kubernetes as a Service on Azure, but unfortunately at the time of writing (Mar 18) we are not able to provision AKS into a custom VNET. At this time, AKS creates it own VNET using `10.0.0.0\8`, which isn't great at this overlaps with the migration network.

Provisioning Azure Container Service via the portal also has the same limitation so I needed something else. Luckily the container folks @ Microsoft have created the [ACS-Engine](https://github.com/Azure/acs-engine), which is an open source project that creates ARM templates to deploy different orchestrators to Azure.

I experimented with a number of options, but the below gist was the end result. Make sure you know the different network policies and also the CIDR mappings for the K8s networks. There is a section in the docs that state the VNET must be in the same Resource Group as the ACS cluster. We did look at a "workaround" this but ended up giving the ACS Service Principal `Owner` rights on the subscription, which is not something I would let go into production!!

<script src="https://gist.github.com/msimpsonnz/c5d074607e119fc0fc531312c9a6461d.js"></script>