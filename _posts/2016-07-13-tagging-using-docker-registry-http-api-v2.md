---
layout: post
title: Tagging images using Docker Registry HTTP API v2
comments: true
---

## Problem

Docker Registry HTTP API v2 is incompatible with v1 and there's no dedicated method to tag an existing image. Instead you should deal with image _manifests_.

## Solution

There are 2 manifest _schemas_ in v2 API:

- _version 1_ which requires to use signed manifests
- _version 2_ (available in Docker Registry >= 2.3) which allows you to use non-signed ones

Working with version 2 is much simpler and that's why we're going to use it.

The idea is very simple:
- download a manifest of an image
- upload the same manifest under a new tag

## Step-by-step guide

- Prerequisites:

```sh
$ docker --version
Docker version 1.10.3, build 9419b24-unsupported
$ registry --version
registry github.com/docker/distribution v2.4.0+unknown
```

- Start registry 2.4:

```sh
$ sudo registry serve /etc/docker-distribution/registry/config.yml
```

- Install an image in the registry:

```sh
$ sudo docker pull busybox
$ sudo docker tag busybox localhost:5000/mybusybox
$ sudo docker push localhost:5000/mybusybox
```

- Get manifest of the uploaded image:

```sh
$ curl 'http://localhost:5000/v2/mybusybox/manifests/latest' \
-H 'accept: application/vnd.docker.distribution.manifest.v2+json' \
> manifest.json
```
Result from `manifest.json` (notice that `schemaVersion` is 2):

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
$ curl -XPUT 'http://localhost:5000/v2/mybusybox/manifests/new_tag' \
-H 'content-type: application/vnd.docker.distribution.manifest.v2+json' \
-d '@manifest.json'
```

- Check that the new tag has been created:

```sh
$ curl 'http://localhost:5000/v2/mybusybox/tags/list'
{"name":"mybusybox","tags":["latest","new_tag"]}
```

## Known issues

Docker pre-1.10 generates manifests only using schema 1. So if you push an image, it's going to have only that manifest. And if you try to retrieve the manifest of schema 2 via curl, `accept` header will be effectively ignored and you'll always get back manifest of schema 1. That's because the registry (at least 2.4) does not support conversion from schema 1 to 2.
