---
layout: post
title: Tagging using Docker Registry HTTP API v2
comments: true
---

## Problem

Docker Registry API v2 is incompatible with v1 and there's no dedicated method to tag an existing image. Instead you should deal with image _manifests_.

## Solution

There are 2 manifest _schemas_ in v2 API:

- version 1 which requires to use signed manifests
- version 2 (available in Docker Registry >= 2.3) which allows you to use not-signed ones

Working with version 2 is much simpler and that's why we're going to use it.

The idea is very simple:
- download a manifest of an image
- upload the same manifest under a new tag

## Step-by-step guide

- Start registry 2.4: `docker run -p 5000:5000 registry:2.4`

- Install an image in the registry:

```sh
docker pull busybox
docker tag busybox localhost:5000/mybusybox
docker push localhost:5000/mybusybox
```

- Get manifest for the uploaded image:

```sh
curl 'http://localhost:5000/v2/mybusybox/manifests/latest' \
-H "accept: application/vnd.docker.distribution.manifest.v2+json" \
> manifest.json
```
Result (`manifest.json`):

```json
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/octet-stream",
      "size": 1459,
      "digest": "sha256:2b8fd9751c4c0f5dd266fcae00707e67a2545ef34f9a29354585f93dac906749"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 667590,
         "digest": "sha256:8ddc19f16526912237dd8af81971d5e4dd0587907234be2b83e249518d5b673f"
      }
   ]
}
```
- Upload the manifest under a new tag:

```sh
'http://192.168.99.100:5000/v2/mybusybox/manifests/new_tag' \
-H "content-type: application/vnd.docker.distribution.manifest.v2+json" \
-d '@manifest.json'
```

## Known issues
