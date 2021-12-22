---
title: "Cost of microservices"
author: Kristoffer Lind
description: "Many of the costs associated with a microservice architecture are often overlooked, this post tries to reason about the pros and cons of picking it."
categories:
- Development
tags:
- architecture
- microservices
date: "2021-12-22"
---

Microservices are great; they're fun to build and run. You'll have lots of interesting new problems to tackle. That does however also mean that you'll need to be reaping quite a bit of benefits from it for it to be a reasonable business choice. So what are the costs? what are the benefits?

# Interesting problems
The associated technical problems are very interesting, which will likely help with both recruitment and keeping those developers from being bored out of their minds and quitting (unless you have interesting business problems).

New technical problems include:
- reliable communication between services
- reliably running and securing multiple services
- learning the orchestration tool chosen for running those multiple services (kubernetes?)
- knowledge sharing (will you be needing a developer portal? a service catalog?)
- shared build and release pipelines
- shared artifacts (hardened base images for docker images for example, strategy for building them and keeping them up to date)
- shared libraries (and avoiding building a distributed monolith when doing so, which kinds of registries will you be needing?)

## Cost
None of the time spent working on these technical problems creates any business value.

# Organizational scaling
This is the main benefit of microservices. If well executed you can have multiple teams working fairly independently from each other. Development velocity can be kept high as you're sidestepping many of the growing pains.

## Cost
The organization needs to change accordingly, you'll need to figure out which teams will own which parts of the business domains and restructure your organization accordingly. You'll likely also get some of it wrong, which means there's also an organizational maintenance cost. 

You should probably also have one or more teams for the platform, developer productivity and security automation.

# Performance scaling
Enables very fine-grained options for horizontal scaling as every service can have different settings for how it will scale.

## Cost
You'll need to configure the resource needs/limits for each of those services, even if you don't actually need the scaling part yet.

Every service should also have health check endpoints for readiness and liveness to run reliably. What are the criteria for it being alive? ready to handle requests?

If you start off with a modular monolith instead you can extract modules into services and define it per service as needed.

# Architectural isolation
Microservices "enforces" a very high level of isolation. Services don't need to be written in the same language or use the same frameworks. It opens up your recruitment pool as you no longer need to search for developers with a specific set of skills. You will have well defined services that can be developed and released independently of each other.

## Cost
I think these are commonly the main arguments when a microservice architecture is picked for the wrong reasons. I'm not sure I even find them beneficial. If you think the team would be building a big ball of mud spaghetti monolith, do you really think adding the complexity of microservices is a good idea?

You've essentially just increased the amount of ways in which you can shoot yourself in the foot and the bullet is now also hollow point and radioactive. That spaghetti will be shared across services using shared libraries where it's harder to figure out which parts are affected by any change. You'll therefore also need to figure out a strategy for how to test each full configuration of service versions in that distributed monolith.

Interfaces being rigid and services possibly not being in the same language also makes refactoring really hard. You are much more likely to end up with a sensible set of services if they were first just modules in a modular monolith where it's easy to move things around as requirements change or knowledge increases. Well defined modules allows working almost as independently and if you really need it they can also be independent build/deployment units in most languages (at the cost of introducing some of the problems of microservices).

Microservices being the solution to your architectural problems is likely a dream. Don't do it prematurely, wait until you actually need it. For now spend that extra time improving software quality. Focus on creating a high quality modular monolith, learning more about the domain and refactoring as you learn. See if you can figure out a more productive way to make the project interesting.

# Conclusion
Unless you need it for scaling organizationally right now, you're probably better off waiting until you actually need it. If you were ready to spend the extra effort to solve some other problem, figure out what the root problem is and spend the effort there. No premature optimization, extract one or a few modules into services when you are prepared to also give full ownership of those services to a different team, if it's owned by the same team that owns the monolith you've gained very little at a very high cost.
