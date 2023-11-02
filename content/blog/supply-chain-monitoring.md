---
title: "Supply-chain monitoring"
author: Kristoffer Lind
description: "Inventory based security monitoring of your supply chain, in this post we'll automate producing inventory specifications and then utilize those specifications to monitor for known vulnerabilities and license issues."
categories:
- Operations Development
tags:
- security
date: "2023-11-02"
---

Software used in this article is locked to specific versions to increase the chance of it working as specified. Make sure you update all of it if you want to actually start using it after experimenting. I've had this article lying around for a bit before publishing, so versions are pretty outdated even at time of publish.

Automate building inventory specifications for your applications and monitor all those dependencies for known vulnerabilities and license issues. Set and enforce global policies to break builds. In this post we'll cover: 
- containers with apk, deb, rpm, javascript, go, java and ruby dependencies. 
- nuget packages

You can expand on this to cover all software used and produced in your organization.

Build machine images for VMs? produce inventory specifications for them and add checks to enforce policies. Are you using closed source software? appliances? have your vendors produce specifications. Maybe you're also a vendor, produce specifications and make them available to your customers, let them know that you care.

# Setup inventory monitoring
We'll be using [OWASP DependencyTrack](https://dependencytrack.org/), they supply docker images for the software components and there's also a [helm chart](https://github.com/evryfs/helm-charts/tree/master/charts/dependency-track) available as a starting point.

To quickly get started trying it out we'll just start it up locally with docker-compose. Create a docker-compose.yml file with this content:

```yml
# docker-compose.yml
version: '3.7'

volumes:
  dependency-track:

services:
  dtrack-apiserver:
    image: dependencytrack/apiserver
    ports:
      - '8081:8080'
    volumes:
      - 'dependency-track:/data'
    restart: unless-stopped

  dtrack-frontend:
    image: dependencytrack/frontend
    depends_on:
      - dtrack-apiserver
    environment:
      - API_BASE_URL=http://localhost:8081
    ports:
      - "8080:8080"
    restart: unless-stopped
```

Then start it up:
```sh
# -d is detached mode, execute 'docker-compose down' to stop
docker-compose up -d
firefox http://localhost:8080
```

You should now be looking at the login screen for your local instance of DependencyTrack. Log in with admin/admin as credentials and change the password when prompted. Go to administration -> access management -> teams -> automation and make a note of what the api key is, we'll need it for posting SBOM (software bill of materials) files to it's api.

When starting up it will create local mirrors for vulnerability sources, so leave it running for a bit to avoid issues. Leaving it running while populating inventory, configuring policies and trying them out is likely enough, but you can check on the progress in the logs:

```sh
docker-compose logs
```

# Build and populate inventory
We now need to create SBOM files for everything we want to keep track of in DependencyTrack. We'll go through a few of them here and if you need more you can take a look at the [CycloneDX tool center](https://cyclonedx.org/tool-center/). These tools also include options for scanning platforms at runtime for particular cases, which might be an alternative if there's a tool for your platform and you're not interested in blocking stuff earlier in the process.

An important thing to note is that if you have more than one tool producing SBOM specifications you'll need to merge them before uploading. DependencyTrack does not support incremental changes, which makes sense, how on earth would it ever know about removed dependencies otherwise. We'll produce the specification from each tool and then have a later step pick them all up, merge them and upload them to the inventory service.

## Containers
Container scanning with [syft](https://github.com/anchore/syft) identifies OS packages from apk, dpkg, nix and rpm and most popular development ecosystems. They do have an incentive to nudge you into using their paid services, but there's lot's of alternatives and it's a great start in populating your inventory.

```sh
# !/bin/bash

set -eu

IMAGE=$1

SYFT_VERSION="0.33.0"
REPORTS_DIRECTORY="$(pwd)/sbom"
DEPENDENCY_TRACK_URI="localhost:8081"
DEPENDENCY_TRACK_API_KEY="dependencytrack api key"

if [ ! -d "$REPORTS_DIRECTORY" ]; then
  echo "Creating reports directory: $REPORTS_DIRECTORY"
  mkdir -p "$REPORTS_DIRECTORY"
fi

if ! hash syft 2>/dev/null || ! syft version | grep -q "$SYFT_VERSION"; then
  echo "Install image inventory tool"
  wget https://github.com/anchore/syft/releases/download/v${SYFT_VERSION}/syft_${SYFT_VERSION}_linux_amd64.tar.gz
  tar -C ~/bin -zxf syft_${SYFT_VERSION}_linux_amd64.tar.gz
  rm syft_${SYFT_VERSION}_linux_amd64.tar.gz
fi

REPORT_PATH="$REPORTS_DIRECTORY/$IMAGE-syft.json"

echo "Checking image"
syft packages $IMAGE \
  --output cyclonedx-json \
  --file $REPORT_PATH
```

## Nuget packages
Container inventory scanner does not fully support dotnet, so we'll need to set that up individually. [CycloneDX-dotnet](https://github.com/CycloneDX/cyclonedx-dotnet) can build dotnet inventory specifications based on .csproj or .sln files.

```sh
# !/bin/bash

set -eu

SCANNER_VERSION="latest"
SCAN_DIRECTORY="$(pwd)"
REPORTS_DIRECTORY="$(pwd)/sbom"
DEPENDENCY_TRACK_URI="localhost:8081"
DEPENDENCY_TRACK_API_KEY="dependencytrack api key"

if [ ! -d "$REPORTS_DIRECTORY" ]; then
  echo "Creating reports directory: $REPORTS_DIRECTORY"
  mkdir -p "$REPORTS_DIRECTORY"
fi

echo "Retrieve dotnet inventory image"
docker pull cyclonedx/cyclonedx-dotnet:$SCANNER_VERSION

echo "Build dotnet application inventory"
docker run --rm \
  --volume "$SCAN_DIRECTORY":/path \
  --volume "$REPORTS_DIRECTORY":/reports \
  cyclonedx/cyclonedx-dotnet:$SCANNER_VERSION \
    /path \
    --out "/reports" \
    --disable-package-restore \
    --json
```

## Merge and upload inventory specifications
A single specification is required per version of a project. If multiple tools were required to build the specifications the results will need to be merged, otherwise that step can be skipped. Operations on specification files can be done with [cyclonedx-cli](https://github.com/CycloneDX/cyclonedx-cli).

The merged result should then be uploaded to DependencyTrack.

```sh
#!/bin/bash

set -eu

PROJECT_NAME="some-project"
PROJECT_VERSION="some-version"
DEPENDENCY_TRACK_URI="localhost:8081"
DEPENDENCY_TRACK_API_KEY=""

CYCLONEDX_VERSION="0.21.0"
REPORTS_DIRECTORY="$(pwd)/sbom"
INVENTORY_SPECIFICATIONS=$(find $REPORTS_DIRECTORY -type f | awk '{printf "%s ", $0}')

if ! hash cyclonedx 2>/dev/null || ! cyclonedx --version | grep -q "$CYCLONEDX_VERSION"; then
  echo "Install specifications utility"
  wget https://github.com/CycloneDX/cyclonedx-cli/releases/download/v0.21.0/cyclonedx-linux-x64
  mv ./cyclonedx-linux-x64 ~/bin/cyclonedx
  chmod +x ~/bin/cyclonedx
fi

echo "Merge inventory specifications"
cyclonedx merge \
  --input-files $INVENTORY_SPECIFICATIONS \
  --output-file $REPORTS_DIRECTORY/full-inventory.json

echo "Upload results to DependencyTrack"
curl -X "POST" "http://${DEPENDENCY_TRACK_URI}/api/v1/bom" \
     -H "Content-Type: multipart/form-data" \
     -H "X-API-Key: $DEPENDENCY_TRACK_API_KEY" \
     -F "bom=@$REPORTS_DIRECTORY/full-inventory.json" \
     -F "projectName=$PROJECT_NAME" \
     -F "projectVersion=$PROJECT_VERSION" \
     -F "autoCreate=true" \
     -v

echo "Remove merged result to allow reruns"
rm -f $REPORTS_DIRECTORY/full-inventory.json
```

## Utilize the information

You now have a source of information for vulnerabilities and license issues over time and when there's a particularly nasty vulnerability found, you can select that vulnerability and get a list of all your affected software. It's also a great indicator of where to spend some extra effort updating, I suggest adding reviewing this information as an action in one of your weekly or monthly meetings, including picking a target for improvement until the next meeting.

Configure a policy for the worst violations (no copyleft licenses or critical vulnerabilities for example), you can also tag services as public or sensitive for more advanced settings. Use notifications or a blocking build step in pull requests as a countermeasure (be very careful with the noisiness of these).

Ideally you've been careful enough about bringing in dependencies that you can evaluate each vulnerability finding to suppress findings that are false positives or does not affect you and just patch the ones that are left. But I'm guessing you are now looking at hundreds to thousands of instances of known vulnerabilities and can't investigate them individually yet, here are some suggested approaches to start moving to a point where that is an option:
- Automate updating vulnerable dependencies and automate updates where that isn't an option
- Improve automated validation of releases to allow bigger updates to pass through without manual efforts
- Regular targeted efforts to improve these metrics
- Minimize your amount of dependencies to reduce the amount of findings to be considered (do you need to support that format? do you need dependencies to check if that number is even or odd? do you need to generate a spreadsheet for download or would it be better to just supply the data?)
- Sort of the same as above, but switch to minimal distributions for docker images(distroless if possible) and VMs (you probably don't need a window manager or tooling for video playback to run your web service, right?)
