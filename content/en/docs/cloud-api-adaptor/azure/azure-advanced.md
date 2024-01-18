---
title: Advanced Azure Deployment Settings
description: This documentation has advanced settings for Azure deployment
categories:
- docs
tags:
- docs
- caa
- azure
---

## Pod VM Images for Cloud API Adaptor (CAA)

### Build a custom pod VM image

If you have made changes to the CAA code that affects the Pod-VM image and you want to deploy those changes then follow [these instructions](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/azure/build-image.md) to build the pod-vm image.

### Use nightly builds

An automated job builds the pod-vm image each night at 00:00 UTC. You can use that image by exporting the following environment variable:

```bash
SUCCESS_TIME=$(curl -s \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/confidential-containers/cloud-api-adaptor/actions/workflows/azure-podvm-image-nightly-build.yml/runs?status=success" \
  | jq -r '.workflow_runs[0].updated_at')

export AZURE_IMAGE_ID="/CommunityGalleries/cocopodvm-d0e4f35f-5530-4b9c-8596-112487cdea85/Images/podvm_image0/Versions/$(date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$SUCCESS_TIME" "+%Y.%m.%d" 2>/dev/null || date -d "$SUCCESS_TIME" +%Y.%m.%d)"
```

Above image version is in the format `YYYY.MM.DD`, so to use the latest image use the date of yesterday.
