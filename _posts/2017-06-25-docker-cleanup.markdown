---
layout: post
title:  "Docker troubleshooting"
date:   2017-06-25 14:17:06 +0530
category: containers
comments: true
canonical_url: containers/docker-troubleshooting.html
tags: [kubernetes, docker]
shortinfo: This post list down issues faced in running docker and steps to resolve issues.
order: 100
---

## Dangling docker images
Dangling images are images that are not referenced by any image or containers. We should remove these images to reclaim space consumed by these images. Caution: dont use below command while `docker build` is in progress.
```bash
docker rm -v $(docker ps -q -f status=exited)
```

## Dangling docker volumes
While using data volume containers, you have to remove container with -v flag as `docker rm -v`. If you don't use the -v flag, the volume will end up as a dangling volume and remain in to the local disk.
```bash
docker volume rm $(docker volume ls -qf dangling=true)
```

## Removing docker containers
We should remove dead, long time exited containers.
```bash
# removing dead containers
docker rm -f -v `docker ps -aq -f status=dead`

# removing exited containers
docker rm -f -v `docker ps -aq -f status=exited`

# removing containers exited atleast 24 hours before
docker rm -v `docker ps -a | grep Exit | grep days | awk '{ print $1 }'`
```
