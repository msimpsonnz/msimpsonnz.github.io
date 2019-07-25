---
layout: post
title: Azure Media Services v3 with Event Grid
summary: Walk through of using Azure Media Services and its recent support for Event Grid to create a event driven workflow from uploading a blob to encoding a video
---

## Walk through using the new Azure Media Services v3 API to create a workflow

Ok, so I need to create Azure Media Services account and a Service Principal to use for auth.

> az group create --name AMS --location centralus

```bash
az storage account create --name mjsdemostrams \  
--kind StorageV2 \
--sku Standard_RAGRS \
--resource-group AMS
```

> az ams account create --name mjsdemoams --resource-group AMS --storage-account mjsdemostrams

> az ams account sp create --account-name mjsdemoams --resource-group AMS

```json
{
  "AadClientId": "00000000-0000-0000-0000-00000000",
  "AadEndpoint": "https://login.microsoftonline.com",
  "AadSecret": "00000000-0000-0000-0000-00000000",
  "AadTenantId": "00000000-0000-0000-0000-00000000",
  "AccountName": "mjsdemoams",
  "ArmAadAudience": "https://management.core.windows.net/",
  "ArmEndpoint": "https://management.azure.com/",
  "Region": "Central US",
  "ResourceGroup": "AMS",
  "SubscriptionId": "00000000-0000-0000-0000-00000000"
}
```
## Azure Function to process Encoding Job

I created a new [repo](https://github.com/msimpsonnz/msft-misc/tree/master/MediaServices.Demo) that has a Function with Event Grid trigger from a blob that queues up a Media Services Encoding job.


## Event Grid for Blob

Now I need an Event Grid subscription to to trigger an Azure Functions to kick of the encoding.

#### Storage Account Function
```bash
az storage account create --name mjsdemoamsfuncstr \
--sku Standard_LRS \
--resource-group AMS
```

#### Create a Function App
```bash
az functionapp create --resource-group AMS --consumption-plan-location centralus \
--name mjsdemoamsfunc --storage-account mjsdemoamsfuncstr  
```

#### Create a EventGrid Subscription
```bash
storageid=$(az storage account show --name mjsdemocdnstr --resource-group CDN --query id --output tsv)
endpoint=$endpoint

az eventgrid event-subscription create \
  --resource-id $storageid \
  --name mjsdemoeventBlob2Func \
  --endpoint $endpoint
```

#### Create a MSI for Azure Functions
>az webapp identity assign --name mjsdemoamsfunc --resource-group AMS
```json
{
  "principalId": "00000000-0000-0000-0000-00000000",
  "tenantId": "00000000-0000-0000-0000-00000000",
  "type": "SystemAssigned"
}
```

#### Add MSI to as Media Services Contributor
>az role assignment create --assignee 00000000-0000-0000-0000-00000000 --role Contributor --scope /subscriptions/00000000-0000-0000-0000-00000000/resourceGroups/AMS/providers/Microsoft.Media/mediaservices/mjsdemoams

## Uploading files via CDN

Ok, so now I have the workflow linked up, time to upload some files. I figured I wanted a single Azure Media Service account but want my users to be able to upload their files quickly. Azure CDN can help here, I can create a CDN endpoint in each of the main regions which proxies back to a storage account back to the same region as AMS, this way I can save on region to region bandwidth but also give a good experience to my users.

```bash
curl -X PUT -T ./iOS.mp4 -H "x-ms-date: $(date -u)" -H "x-ms-blob-type: BlockBlob" "https://mjsdemocdnstr.blob.core.windows.net/raw/iOS.mp4" -w "%{response_code};%{time_total}" > dataFile.txt 2> informationFile.txt
```
*24.639 seconds

```bash
curl -X PUT -T ./iOS.mp4 -H "x-ms-date: $(date -u)" -H "x-ms-blob-type: BlockBlob" "https://mjsdemocdn.azureedge.net/iOS.mp4" -w "%{response_code};%{time_total}" > dataFile.txt 2> informationFile.txt
```
*18.746 seconds

So about a 30% improvement, I am sitting in New Zealand the storage account in the first command is in 'US Central' and the CDN endpoint in the second command is in 'Australia East'