---
title: "Security automation"
author: Kristoffer Lind
description: "Security automation overview, automating dependency scanning, container scanning, source code scanning, IDE based checks and penetration testing."
categories:
- Operations Development
tags:
- security
date: "2021-12-23"
---

A typical web application involves a lot of code in many layers and manually keeping it secure requires in depth knowledge in all of those layers and lots of maintenance. To increase your chances of keeping it secure you could utilize automation. There are a lot of tools available for helping with security automation and it's a pretty big task to sift through them to find good ones. There are lots and lots of companies selling security tools who wants you on their hook. Incentive is therefore an important aspect in picking.

This post will try to guide you through which tools are currently available, how to use the ones I picked along with some advice on processes and related issues. The following types of scanning will be discussed:
- application dependency scanning
- source code scanning
- IDE based checks
- docker container scanning
- automated penetration testing
- infrastructure as code scanning

We'll start off looking at a setup that uses only cli based checks in your existing CI/CD pipelines and forces action by breaking builds. These checks provide a lot of value for the effort but requires a pretty high signal to noise ratio to avoid causing too much frustration. To keep that up it needs to be easy to suppress warnings and it needs to skip rules or parts with a high rate of false positives.

In an upcoming post we'll look at a monitoring approach that will include running services and databases to allow keeping track of issues across projects. If you have the needs/resources to keep track of those dashboards and drive engagements it might be better to start there.

Taking a hybrid approach isn't a bad idea if you want monitoring. Enforce some actions using this approach and keep track of all the issues using monitoring. Findings from monitoring can be used to tune enforcement and drive engagements.

# Dependency scanning
[OWASP dependency-check](https://owasp.org/www-project-dependency-check/) is a wrapper around most language/platform specific dependency scanners. If you want more control over how the scanning works it's easier to just configure the underlying analyzers. It's a great starting point and you can disable analyzers and set them up individually as needed.

```bash
#!/bin/bash

set -eu

VERSION="latest"
PROJECT_NAME="my-project"
DATA_DIRECTORY="/cache/dependency-check"
REPORTS_DIRECTORY="$(pwd)/reports"
SCAN_DIRECTORY="$(pwd)"
CVSS_THRESHOLD="5"

if [ ! -d "$DATA_DIRECTORY/cache" ]; then
  echo "Creating data directory: $DATA_DIRECTORY/cache"
  mkdir -p "$DATA_DIRECTORY/cache"
fi

if [ ! -d "$REPORTS_DIRECTORY" ]; then
  echo "Creating reports directory: $REPORTS_DIRECTORY"
  mkdir -p "$REPORTS_DIRECTORY"
fi

if [ ! -f "$SCAN_DIRECTORY/dependency-check-whitelist.xml" ]; then
  echo "Create empty whitelist"
  cat >"$SCAN_DIRECTORY/dependency-check-whitelist.xml" <<-DC_WHITELIST
    <?xml version="1.0" encoding="UTF-8"?>
    <suppressions
      xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd"
    >
    </suppressions>
  DC_WHITELIST
fi

echo "Ensure we have the latest version of dependency check"
docker pull owasp/dependency-check:$VERSION

docker run --rm \
  -e user=$USER \
  -u $(id -u ${USER}):$(id -g ${USER}) \
  --volume "$SCAN_DIRECTORY":/src \
  --volume "$DATA_DIRECTORY":/usr/share/dependency-check/data \
  --volume "$REPORTS_DIRECTORY":/report \
  owasp/dependency-check:$VERSION \
    --scan /src \
    --format "ALL" \
    --failOnCVSS "$CVSS_THRESHOLD" \
    --project "$PROJECT_NAME" \
    --out /report \
    --suppression "/src/dependency-check-whitelist.xml"
```

## NodeJS
Most NodeJS projects have thousands of dependencies, often most of what's considered the development environment for other languages is part of this installation. The result of this is that scanning will be very noisy, producing a massive amount of false positives, often for dependencies that are only used during development, but would be a problem if actually deployed.

Create-react-app is a very popular approach for starting react projects and a freshly created project using that will currently result in about 1700 dependencies. While that sounds crazy, consider that it includes a lot of what dotnet developers are getting in their 20+ GB visual studio installation. Some if it is down to philosophies like DRA (don't repeat anyone), really small packages, download padding and deep nesting. But let's pop back out of that rabbit hole.

If you have projects that look like that, you can try this approach. Development dependencies should be code utilized only during development, that is not included in the built bundle. Therefore you probably only care about vulnerabilities designed to attack development machines and build agents during installation or build phases among those dependencies. Those vulnerabilities are likely to be listed as critical, ignoring the rest will keep noise at a minimum.

Also, noting that this is a thing should make you consider doing those installations a lot more isolated from now on. Anyways, we can therefore split the check like this:

```sh
echo "Checking dependencies for moderate or worse vulnerabilities"
npm audit --only=prod --audit-level=moderate

echo "Checking devDependencies for critical vulnerabilities"
npm audit --only=dev --audit-level=critical
```

That works if you can enforce fixing vulnerabilities or moving them to development dependencies if that's where they belong. You do however likely want to be able to whitelist vulnerabilities that don't affect you, which is sadly not supported by `npm audit` (has been requested for years). I've used [audit-ci](https://www.npmjs.com/package/audit-ci) as an alternative, but check if it's supported by npm audit yet or consider alternatives. Doing it based on json output from npm audit is also an alternative.

Configuration file example for audit-ci
```js
{
  "allowlist": [
    1,
    2
  ]
}
```

```sh
echo "Checking dependencies for moderate or worse vulnerabilities"
npx audit-ci --moderate --skip-dev --config audit-ci.production.json

echo "Checking devDependencies for critical vulnerabilities"
npx audit-ci --critical --config audit-ci.development.json
```

Note that it includes an example of using different configuration files for development and production dependencies. You can't put comments along with the whitelist to explain why they were put there and the ids in the allow list are not the searchable advisories. Those are the main reasons I'm not too fond of it. I'd also suggest doing an `npm audit` separately in there just for the output (might be a personal thing, but I find that report easier to sift through).

You could also choose to rely on antivirus and developers keeping their eyes open when doing installs for the development dependencies and just skip them. In that case you can stick with just dependency check by adding the `--nodePackageSkipDevDependencies" argument to dependency-check.

There is a bit of risk involved with not fully scanning development dependencies though. It's not unheard of for the sorting to be incorrect for example (it's hard to keep track of). But the workarounds that will happen as a result of the frustration will be worse.

# Code scanning
These tools definitely will not find everything, but they're constantly improving and some help is better than none. Sadly I couldn't find any wrapper tool like the one for scanning dependencies, but [semgrep](https://semgrep.dev) is a pretty generic tool that even has it's own rules registry. I'd prefer recommending something without commercial intentions, but as long as you avoid spending too much effort on custom rules you can just switch if it gets too bad. Link all invocations to a script like the one below and it should be easily replaceable.

OWASP maintains a list of [source code analysis tools](https://owasp.org/www-community/Source_Code_Analysis_Tools), I'd suggest looking there for alternatives.

Semgrep has two recommended starting sets of rules, one intended for full automation that only contains high confidence rules and another one intended to be used for manual review. Put the first one in CI checks that trigger per commit and the other one in a pipeline that runs on a schedule. Don't forget to also set aside some time to look at the reports of those scheduled runs. You could start off by triggering it once per month and booking a calendar slot for looking at it later that day.

CI check:
```sh
#!/bin/bash

VERSION="latest"
SCAN_DIRECTORY="$(pwd)"

echo "Ensure we have the latest version of semgrep"
docker pull returntocorp/semgrep:$VERSION

echo "Scan code in $SCAN_DIRECTORY using semgrep"
docker run \
  -v "$SCAN_DIRECTORY":/src \
  --workdir /src \
  returntocorp/semgrep:$VERSION \
    --metrics off \
    --strict \
    --config=p/r2c-ci
```

You can explore the semgrep registry [here](https://semgrep.dev/explore), notice how `--config=p/r2c-ci` matches the name of the recommended getting started set r2c-ci. The only difference in that script for running the noisy ruleset is switching r2c-ci to r2c-security-audit. You can also pick multiple rulesets by having multiple `--config` arguments.

They also have an alternate version that supports scanning only the diff against master/main/trunk branch, but it has a proprietary license, look [here](https://github.com/returntocorp/semgrep-action) for more details.

I'd suggest also configuring linter based security tools for your main languages. Those issues can then be addressed while writing code, making the feedback loop a lot shorter.

## NodeJS
You could scan NodeJS specifically using njsscan, based on a quick check though the same rules are available in semgrep registry and the tool is basically a prepackaged way of running semgrep configured to scan NodeJS projects. I'd suggest just configuring the rules you care about in semgrep, that way you can run the same scan for all your projects (unless all your projects are NodeJS based, in which case this solution might be easier).

## Dotnet
Support for dotnet projects is pretty new in semgrep currently, which means there are very few rules in the registry. Unless there are now more available rules for dotnet in semgrep, I'd recommend [Security Code Scan](https://security-code-scan.github.io/) for scanning dotnet projects. Actually since it can also be shown as part of intellisense I'd recommend adding it anyway.

There are three different ways it can be used:
- standalone
- visual studio extension
- nuget package

Including it as a nuget package sets it up as both a development tool (intellisense) and build time check, so that's probably the best choice. It can be installed for a project like this: `Install-Package SecurityCodeScan`.

It does require quite a bit of effort to set it up for each project so if you have a lot of projects you can run it standalone like this:
```sh
dotnet tool install --global security-scan
security-scan /path/to/solution.sln
```

During build issues will show up as build warnings, so you'll need to run the build with warnings as errors to fail builds. Warnings can be [suppressed](https://docs.microsoft.com/en-us/visualstudio/code-quality/in-source-suppression-overview?view=vs-2022) like any other build warnings. If the project has many other warnings it can be tedious to work through them, but it's probably a good idea to look at them.

```sh
dotnet build -warnasserror some/project.csproj
```

## Other code scanning tools
- [GitLeaks](https://github.com/zricethezav/gitleaks) Prevent git leaks
- [license_finder](https://rubygems.org/gems/license_finder) Enforce license requirements
- [GoSec](https://github.com/securego/gosec) Golang
- [brakeman](https://brakemanscanner.org/) Ruby on Rails
- [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF) Android/iOS
- [flawfinder](https://dwheeler.com/flawfinder/) C/C++
- [sobelow](https://github.com/nccgroup/sobelow) Phoenix

# IDE based checks
Linter based security checks can be triggered while writing or on save, which results in a very short feedback loop. These are hopefully going to be fixed before even ending up in a commit. Set these up as a recommended development tool, configured as part of the project if possible and include a check for them during build in your CI server.

Linting is also a very good way to document and enforce code style conventions, so that you can stop worrying about those parts in code reviews, hopefully leading to higher quality reviews.

## NodeJS
ESLint is so popular I'm going to assume it's already installed, if for some reason it isn't, it's fairly simple and you could start off with only these rules. Add [eslint-security-plugin](https://github.com/nodesecurity/eslint-plugin-security) like this:

Add dependency: `npm install --save-dev eslint-security-plugin`  
Configure (add to .eslintrc):
```js
"plugins": [
  "security"
],
"extends": [
  "plugin:security/recommended"
]
```

There are plenty of other very specific ESLint plugins that include security checks, but they're likely already included in the recommended configuration for that specific framework/library. If it's not already configured, see if you can find one for the framework used in that project.

Also make sure it's done as part of the build.  
In package.json:
```json
{
  "scripts": {
    "lint": "eslint src"
  }
}
```
Then make sure `npm run lint` happens at some point during build, it will allow warnings by default, so that you can have really picky stuff or new rules as warnings and enforced rules as errors.

## Dotnet
Install nuget package version of Security Code scan as described above.

## Python
I've never done any python development, so I'm not clear on what's best practice, but [bandit](https://github.com/PyCQA/bandit) looks like a really nice tool. You can install it with: `pip install bandit` and then activate it in vscode by installing the python extension (ms-python.python) and adding the following configuration in .vscode/settings.json:
```json
{
    "python.linting.banditEnabled": true,
    "python.linting.enabled": true
}
```

You can add it as a CI check like this:
```bash
pip install bandit
bandit -r path/to/code
```

## Other lint/IDE based tools
- [spotbugs](https://spotbugs.github.io/)+[find-sec-bugs](https://find-sec-bugs.github.io/) Java security checks
- [PHP CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer)+[phpcs-security-audit](https://github.com/FloeDesignTechnologies/phpcs-security-audit) PHP security checks

# Container scanning
Most docker images are full of known vulnerabilities even when fresh, with new ones being added daily. If you've gone for default images (usually debian based), this scan is going to find so many vulnerabilities that it's frustrating to keep track of. Switch to images that have less stuff you're not using anyway included. Most of the time there's an alpine version available, which has way less vulnerabilities to keep track of as there's just less stuff included.

[Trivy](https://github.com/aquasecurity/trivy) can be used to scan docker images and Dockerfiles. Image scanning supports both giving it an image name and filesystem scan.
Your hardening process might prevent image scan from working on the final image, for example if you're removing the package manager. If that's the case you can utilize the filesystem scan to do it a bit earlier in the process when it still works.

```bash
# !/bin/bash

IMAGE=$1

TRIVY_VERSION="0.21.2"

if ! hash trivy 2>/dev/null || ! trivy --version | grep -q "$TRIVY_VERSION"; then
  echo "Install image scanning tool"
  wget https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
  tar -C ~/bin -zxvf trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
  rm trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
fi

echo "Checking image"
trivy image --vuln-type os --exit-code 1 $IMAGE
```

Trivy has started including a bunch of other checks for library packages and configuration. specifying image scan and only os packages lets us use other tools for those parts.

Whitelisting vulnerabilities is done with a .trivyignore file like this:
```sh
# some reason for whitelisting
CVE-XXXX-XXXX
```

# Infrastructure as code scanning
[kics](https://kics.io) likely handles all your IaC scanning needs, it can find configuration issues with most IaC tools. You should scan all infrastructure repositories, less obvious is that you should likely also scan all application directories for Dockerfile and Helm issues.

```bash
#!/bin/bash

set -eu

SCANNER_VERSION="latest"
SCAN_DIRECTORY=$(pwd)
REPORTS_DIRECTORY="$(pwd)/reports"

if [ ! -d "$REPORTS_DIRECTORY" ]; then
  echo "Creating reports directory: $REPORTS_DIRECTORY"
  mkdir -p "$REPORTS_DIRECTORY"
fi
tf
echo "Ensure we have the latest version of IaC scanner"
docker pull checkmarx/kics:latest

echo "Execute IaC scan in $SCAN_DIRECTORY"
docker run --rm \
  -u $(id -u ${USER}):$(id -g ${USER}) \
  --volume "$SCAN_DIRECTORY":/path \
  --volume "$REPORTS_DIRECTORY":/reports \
  checkmarx/kics:$SCANNER_VERSION scan \
    --path "/path" \
    --output-path "/reports" \
    --report-formats "html,json,sarif" \
    --output-name "kics-results" \
    --fail-on "high" \
    --exclude-categories "Best Practices"
```

False positives can be ignored with a comment:
```terraform
# kics-scan ignore-line
# which ignores that line and the one after, so put it on it's own on the line before
```

We've ignored the best practices category, those findings are better handled using a linter that can be configured to show warnings within IDE.

# Penetration testing
[Owasp ZAP](https://www.zaproxy.org/) is a nice tool for automating application penetration testing. They provide great starting points for penetration tests of web applications and apis. If you want more there's a pretty good ecosystem around it and you can add new scenarios.

It's rules are divided into passive and active, passive is read only while active actually tries sending payloads. Next steps are under the assumption that you own the application you're testing.

For a public web application it's rather easy to get started, you can run a scan with just this command.
```sh
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  --target https://www.example.com
  #-j # ajax spidering, include for javascript based applications
```

For an API with an OpenAPI spec (swagger for example) or GraphQL endpoint with introspection allowed:
```sh
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  --target https://www.example.com/swagger.json
```

Last time I tried the GraphQL one the spidering ended up being a massive list of endpoints and it could be the same for both application scan and API scan. Unless that list is massive though it's fast enough for running as part of every deploy. You could do it as part of build by running it against a local version. Running it after a deploy though means you're also testing the infrastructure, it probably happens a bit less frequently and the odds of someone actually waiting for it to complete are a bit lower.

We do want to configure rules, generate reports and generally tune things a bit more though so let's dig a little deeper.

```sh
# zap-rules.config
*       FAIL
xxxxx   WARN
*       OUTOFSCOPE  regex of stuff to skip
```

Default to failing on all rules, add WARN, IGNORE or OUTOFSCOPE to tune. You can also have a progress file for issues that have already been placed in your issue tracker.

```sh
# zap-progress.json
{
	"site" : "www.example.com",
	"issues" : [
		{ 
			"id" : "10016",
			"name" : "Web Browser XSS Protection Not Enabled",
			"state" : "inprogress",
			"link": "https://www.example.com/bugtracker/issue=1234"
		}
	]
}
```

To execute an API scan with configuration included (application scanning is nearly identical and therefore skipped).

```sh
#!/bin/bash

set -eu

SCANNER_VERSION="latest"
CONFIGURATION_DIRECTORY="$(pwd)"
REPORTS_DIRECTORY="$(pwd)/reports"
TARGET_URL="https://www.example.com/swagger.json"

echo "Ensure we have the latest version of zap"
docker pull owasp/zap2docker-stable:$SCANNER_VERSION

echo "Scan code in $SCAN_DIRECTORY using zap"
docker run --rm \
  -u $(id -u ${USER}):$(id -g ${USER}) \
  -t \
  --volume "$CONFIGURATION_DIRECTORY":/configuration \
  --volume "$REPORTS_DIRECTORY":/reports \
  owasp/zap2docker-stable:$SCANNER_VERSION zap-api-scan.py \
    -r /reports/zap-report.html \
    -J /reports/zap-report.json \
    -c /configuration/zap-rules.config \
    -p /configuration/zap-progress.json \
    -I \
    -t $TARGET_URL
```

You may also want to include a context configuration file for specifying how to authenticate and which urls should be included in scope for the scanning. Configure context using ZAP desktop and pass it in like this:

```sh
-n /configuration/zap-context.config
```

# Signal to noise ratio
The idea with all checks above is to stop the worst issues by breaking builds. It's important to tune it so that it doesn't become a source of frustration. I've tried to define good starting options but the risk vs frustration tradeoff is going to be different for every project.

It's also important that there's always escape hatches, a defined way of suppressing a warning, which I hope I've defined reasonably for all of them. You'll also need a strategy for what's supposed to happen when something is suppressed. If it's an obvious false positive it can just be suppressed, if it can be quickly fixed then just do that. But for more complicated cases you might want to create an issue in your issue tracker for every suppression. Added suppressions should be carefully reviewed as part of code review.
