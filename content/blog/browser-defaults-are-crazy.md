---
title: "Browser defaults are crazy"
author: Kristoffer Lind
description: "Browsers default to allowing js and communication with everything that is accessible from your machine, I think that's crazy and the source of quite a lot of issues with the web. Here's some pointers on why and what you can do about it."
categories:
- security
tags:
- security
- privacy
date: "2023-11-09"
summary: "I could use you as a bot to assault web services I dislike, utilize some of your compute to calculate stuff I'm interested in or mine cryptocurrencies. I could use it to gather enough information about you that I can recognize you from a different browser or private mode. I could run P2P (peer to peer) software to have you help serve content I want to spread. Oh and how nice of you to leave this tab open for a while before you had time to read it, right?"
---

<p>
  <script>
    document.write("Thank you for entrusting me with all these powers, was it a conscious decision? Let me explain why I think those defaults are bad and what you can do about it.");
  </script>
  <noscript>
    Well done, this site works just fine without javascript, being conscious about when to allow it means you are a lot less susceptible to all of the things mentioned below.
  </noscript>
</p>

I could use you as a bot to assault web services I dislike, utilize some of your compute to calculate stuff I'm interested in or mine cryptocurrencies. I could use it to gather enough information about you that I can recognize you from a different browser or private mode. I could run P2P (peer to peer) software to have you help serve content I want to spread. Oh and how nice of you to leave this tab open for a while before you had time to read it, right?

What can be done with it is beautiful, it allows us to build really interesting stuff to publish as web apps that require no installation, with most of the capabilities of a natively installed app. Beautiful things like end to end encrypted decentralized applications that are kept running because people want to use them, where as a user you're part of the system, own your content and who to share it with. 

Most of the time it's used rather wastefully from a user perspective though and the fact that we default to allowing it is insane. We should be giving away that power on a per-domain or per-subdomain basis, where reasonable.

I'm thinking the considerations should look something like this:
- A game, I can see how its needs are a bit different and I don't do banking or other sensitive tasks on this device/compartment.
- An article and the screen is blank without it, I'd rather just look for the information somewhere else than trust this random person.
- A collaborative drawing tool?, that's interactivity sort of on par with a game, seems like a reasonable request, should I trust whoever made it? on this device/compartment?

The main part of it is that we allow JavaScript by default, but it's also about the fact that it's not isolated to the domain you're visiting. I would only be able to attack my host and your machine if it was isolated to this domain. With current defaults though I can attack anything that is accessible from your machine and all of it is also available for help with an attack on you.

## Digging a bit deeper

This section goes a bit deeper on the problems with these defaults. You can scroll past it if you just want to see suggestions for improvement.

### Ephemeral bot network

While having a browser tab of a web service open, you could be part of what's effectively an ephemeral bot network, the site in question is the command server instructing you what to do. Each visitor is an ephemeral bot of that network that can be utilized for compute power, network diversity and bandwidth. It can also be a bit more targeted, utilizing network access given to those particular machines or other web services those clients are authorized in. It can also be used for mass content creation like reviews from seemingly random users.

You might be thinking that it's only a problem if visiting shady places, but if you consider the monetization perspective that doesn't make sense. Most attacks are more likely to be designed to be as broad as possible, utilizing automation. Malicious actors are continuously scanning the web for vulnerable services, some of which are attacked right away while others are likely classified as potentially juicy or just don't have an attack strategy implemented yet and are saved for later use. Now that tab you've left open for a menu of the local restaurant might be making you part of such a bot network.

#### Compute

With access to WebAssembly and WebGPU this distributed compute power can be utilized pretty efficiently. Some cryptocurrencies are also moving towards making specialized mining hardware less efficient. One approach for monetizing such a network could therefore be to utilize it for mining cryptocurrencies.

#### Attack platform

The network diversity/bandwidth could be utilized for a DDOS (distributed denial of service) attack. Such an attack could be made very hard to distinguish from regular use.

Network access given to those particular clients could also be utilized as a stepping stone to gather information from or attack services or servers of corporate/organization networks.

#### Content spreading

Using P2P technology your bandwidth can help with spreading content to other peers. Scary as it might sound that I could have you help distribute something of my choosing, I have a hard time finding malicious monetization strategies. Maybe it could be used as a covert channel of spreading malware or possibly used in a sequence, by having you spread something illegal this way and then use evidence of that to blackmail you.

### Malware distribution

It's not uncommon for malware to be distributed using ad networks. It's in fact so common that blocking ads has been recommended by government agencies as a security measure. Hacking CMSes or injection attacks are also utilized for distribution. The actual malware is commonly loaded from other sources, which is then happily fetched and executed according to our default settings.

By requiring an establishment of trust before giving away these extended powers, malware would need to be distributed through trusted sources to be as effective. That in turn should also increase legal implications as it's no longer an indirect distribution; incentivizing being a bit more careful. Which should make distribution harder.

### Information leakage

The most common use case is tracking, which is not quite as sinister, but probably not something you actually want. Most services are going to load scripts from a couple different companies with the aim of building a profile of you that is as specific as possible. Mostly because the more detailed they can make this profile; the more money they can charge for showing you ads. This includes keeping track of how long you spend on every part of every page, what you've searched for, what you liked/disliked, what you put in a cart somewhere or what you bought, along with fingerprinting and other techniques to merge this data with that of your other devices or from other services.

Maybe that's fine; as a people we've sort of voted for the alternatives where we are the product. But what happens when this data leaks to more sinister actors? All it takes is a mistake in configuration somewhere or a malicious/blackmailed employee with access to these systems. It will leak, to some extent it has already happened, it just hasn't had a catastrophic enough result yet. What could a more sinister actor do with this?

What if your government turns into a dictatorship? You could now be punished for having shown interest in a religion, other competing political powers or because you like to eat sandwiches in the morning and your dictator is particularly crazy. 

What if it's accessible for organized crime? How could the information be monetized in darker ways? Parts of it are already being bought/sold, but for businesses it's too profitable to be an information center, so they're protective of it to some extent, perhaps the difference is that it would start being sold as complete, merged profiles without any regard for who the buyer is. 

Those profiles could then be used for identity theft, phishing and blackmail, just to name a few I can think of. Identity theft could be so well targeted that it could prove difficult to clear yourself of crimes committed with it. Phishing campaigns could essentially be very broad and intelligent spear phishing campaigns.

## What can you do about it?

Default to using a more restricted browser, just picking one of the popular ones with [NoScript](https://noscript.net/) extension installed is a good option. With that extension JavaScript and other potentially harmful content is disabled by default and permitted on a per subdomain basis. Pretty much modifying browser defaults to what I think they should be. Most of the time it's just more efficient, you can read the content you're interested in and just ignore tons of scripts, most of the "Would you like to be spied on?"-dialogs are loaded from external sources so you don't even see them. In cases where you don't just want to ignore badly designed sites though it can be a bit frustrating, resources are loaded from 10-15 different places and you need to figure out which 3 of them are part of loading the content you're interested in.

Using different browsers for different purposes helps. In it's simplest form this is just different browser installs on the same OS. If that's your approach I'd suggest also compartmentalizing it with virtualization to minimize the blast radius when your time comes.

### Web-based is safer than native

The web is considered a rather hostile environment and browser developers do a lot of work to secure and isolate them. It's therefore a more restricted environment with less privileges than natively installed applications. Using a web-based alternative is safer than installing a native alternative most of the time. I think the case for picking native applications is when you want to isolate/own the data or need the extra performance or privileges.

A bit more concretely, none of those apply for most communication tools (like Slack, Teams or Discord), hence I use the web versions of those. For P2P communication tools (like Matrix) where owning and isolating your data is an option however it might be the better option. Password managers and version control systems are also fall into this category. While for a game it's mostly about performance, but also about being able to play it after the company sells it or shuts it down ("owning" it).

### Network-level mitigation

At a network level you could utilize DNS and IP based blocking of malware and trackers. There's tons of regularly updated lists of known malicious services. This type of blocking uses those lists to to redirect all those requests somewhere else. This will help protect all devices on your network and helps with software other than the browser. Do be aware that you might need to jump through some hoops for nasty software that does not respect your advertised or configured DNS settings though.

Your simplest alternative would be to [pick a DNS provider](https://avoidthehack.com/best-dns-privacy) that does the kind of filtering you would like it to. 
If you want more control you can self-host such a service using [pi-hole](https://pi-hole.net/). Or you can utilize the underlying blacklists in your firewall, where you can also include ip based blocking.

These rules can't really be as strict though as they're being applied a lot broader and you probably don't want to have to pop into that UI to allow something temporarily while on your phone for example.

### Compartmentalize

Utilize virtualization to sort your tasks into compartments, managing virtual machines as a way of isolating them from each other. This strategy assumes a breach and instead tries to minimize the blast radius. Use a disposable or stateless VM for default browsing, separate VMs for client work and separate VMs for admin tasks. You can achieve that with any virtualization configuration of your choice, but the following is how I've chosen to do it.

[Qubes OS](https://www.qubes-os.org/intro/) is a meta OS based on that idea, you compartmentalize your applications into different types of qubes (preconfigured virtual machines with varying persistence and settings), run all your applications in them and it mostly looks like a regular window manager with labels and colors to help you keep track of qubes and hence trust levels they belong to. The qubes can run whatever OS you want, so if you need to you can have a minimal Windows (LTSC might be an option as it excludes a bunch of components you probably don't need in this setting) install to run a few applications where you need that and it'll be just another window per application in your window manager.

Tooling for working with and managing these qubes is great and it's a lot more user friendly than it sounds. It's default configuration of qubes is likely a great starting point for you and you can customize it to your liking as you come up with what compartmentalization works best for you. There are a couple of things to get used to when switching:
- copy/paste across qubes adds two steps: ctrl+c -> ctrl+shift+c (copy from vm to host context) -> ctrl+shift+v (copy from host to vm context) -> ctrl+v
- move files across qubes: right click file -> move to other vm (which moves it to QubesIncoming directory of that VM)
- remembering to pass camera, microphone and audio devices to meeting qube
- installations are done in template qubes, not app qubes
- screen sharing is isolated to the qube you're in, unless you add a solution for sharing across qubes [qubes-video-companion](https://github.com/QubesOS/qubes-video-companion)

Once you're comfortable with those it's just another OS and you'll mostly be modifying your structure of qubes, where you want to open links by default (like opening mail attachments in a disposable qube for example) and look into GPG/SSH-split setups to minimize the risk of those keys from being compromised along with the qubes where you use them.

If you're a bit nerdy and being able to configure your OS with code appeals to you; Qubes OS comes with saltstack ready to be enabled and used. I'm experimenting with it and will hopefully have a configuration I can showcase with a post and repository at some point.

Check their [hardware certification requirements](https://www.qubes-os.org/doc/certified-hardware/#hardware-certification-requirements), for a concise description of security considerations regarding hardware and firmware (if you find that interesting you'll be reading linked resources for a while).
