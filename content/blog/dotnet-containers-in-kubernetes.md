---
title: "Running dotnet containers in kubernetes"
author: Kristoffer Lind
description: "Caveats when running dotnet applications in docker images on kubernetes"
categories:
- Development
- Operations Development
tags:
- dotnet
- docker
- security
- performance
date: "2021-12-04"
---

There are some caveats to running dotnet in containers. You might expect that the docker images you pulled from Microsoft's container registry were checked for vulnerabilities and prepared to run in a containerized environment. If you've gone with defaults and don't have strict rules on your cluster you'll have an image configured to run alone on a server (which is not a usual setup in a containerized environment) that is also full of known vulnerabilities even when fresh. To be fair though this is a very common problem with docker images.

# Performance
## Garbage collection
In default configuration these images are going to operate as if they're the only thing running on that node. Garbage collection will not happen until it's using 90% of the available memory. It thinks it has all of the node's memory to itself and garbage collection therefore never happens.

Setting memory limits for the deployment should mean that heap limit is set to 75% of that value. That default value is applied if the process knows it's running in a container and HeapHardLimitPercent is not set. Heap limit can also be configured by setting environment variables `COMPlus_GCHeapHardLimit` (.NET core 3.0 or later) or `DOTNET_GCHeapHardLimit` (.NET 6 or later). Values should be defined as bytes in hexadecimal. Heap limit setting is ignored if Per-object-heap limits are configured.

You might have skipped limits in your production environment to avoid OOMExceptions or CPU throttling. In reality though skipping those limits likely produces more exceptions than having sensible limits set.

# Security
## Known vulnerabilities
Current version of mcr.microsoft.com/dotnet/runtime-deps:6.0 has 62 known vulnerabilities, of which 4 are listed as critical. Latest debian image from which it's derived has 60, with 4 critical. This is a pretty common scenario, most default tags of docker images derive from debian, which is full of known holes from the start. If you've neglected updating the image for a while it'll have several hundred known vulnerabilities.

Do you want to sift through even 60 known vulnerabilities to make sure they don't affect you? No? Neither do I. Maybe the initial ones have been checked, but there's new ones reported daily, do you want to keep track of those?

They also supply alpine versions of their images, these have way less vulnerabilities to keep track of, pick those instead if you can. Current version of mcr.microsoft.com/dotnet/runtime-deps:6.0-alpine has 0 known vulnerabilities. These images also have vulnerabilities reported from time to time, but it's at a level where you can read up on them and decide whether or not to patch.

If you're doing any graphics work (generating excel files for example) and depend on libgdiplus you're stuck with the debian versions, unless you want to include edge testing libs in your production environment.

## Docker configuration
Unless you set a user that application of yours is going to be running as root user. Again, this is very common in the docker ecosystem. The user isn't changed because you might want to install something more. I'd prefer a security first approach where you'd need to switch back to root user for those installs, which would also make it very visible that the root user is active.

Containers aren't magic, you should treat them sort of like any other server OS. That user running the application shouldn't be allowed to do a single thing more than it needs to. It should only have software required for running the application installed. They do however have the benefit that it's really easy to argue for them being immutable. No snowflake pet servers, thank you very much.
