---
title: "Hugo performance optimizations"
author: Kristoffer Lind
description: "Performance optimizations for Hugo themes, image processing, CSS optimizations and prefetch/prerender pages on likely click intent."
categories:
- Development
tags:
- hugo
- performance
date: "2021-11-21"
---

To have a silky smooth start you'll need any code that affects layout to be included in the first render. Ideally you should fit all HTML and CSS needed by first render within 14KB (search for TCP slow start to dig deeper). Size of images needs to be predetermined so that rendering them won't affect layout. There shouldn't be any JS that affect layout either.

# User first/Content first
The reason users come to your site is to consume your content (read your posts, buy your product or whatever). Content is what matters, delivering it should be your primary focus. Your users should be able to start consuming your content as soon as possible, any other priorities about design or analytics are at most secondary. You can make it gradually more beautiful or add functionality. But you should not be disturbing your users when doing so.

# Experience your site in bad conditions
Both Firefox and Chrome have throttling options in the network tab, try setting it to 3G or 4G, whichever is the likely worst condition for your users. Do you feel like ripping your hair out? This is the experience some of your users is going to have.

# CSS optimizations
If your CSS is somewhat slim you can either inline it or utilize server push with a resource hint. If you have lots of CSS you might also want to look into splitting it in modules based on what's needed for different pages and possibly critical/non-critical CSS. Another and perhaps better alternative is to reduce it's size. The size and complexity of your stylesheet also affects parse and render time.

## Inline CSS
Main downside of this approach is caching, it's all going to be downloaded for every single page. On the other hand it's going to work for all browsers and servers. For small CSS files the reduced network overhead, reduced amount of HTML and simplicity makes this the better option. If you can fit both the CSS and a some content within the first TCP roundtrip (14 KB), I think this is the better approach.

```go-html-template
<head>
  {{- $styleOptions := dict "outputStyle" "compressed" }}
  {{- $style := resources.Get "scss/tale.scss" | resources.ToCSS $styleOptions }}
  <style>{{ $style.Content | safeCSS }}</style>
</head>
```

## Resource hint
This approach relies on the server to pick up on the hint and utilize it to trigger push. The browser also needs to support HTTP/2 and accept the push. The server is going to push that resource for every single page so the difference in bandwidth used on the server end is negligible, but the browser can reject it.

```go-html-template
<head>
  {{- $styleOptions := dict "outputStyle" "compressed" }}
  {{- $style := resources.Get "scss/tale.scss" | resources.ToCSS $styleOptions }}
  <link rel="preload" as="style" href="{{$style.Permalink}}">
  <link rel="stylesheet" href="{{$style.Permalink}}" />
</head>
```

## Splitting into parts
In a component based approach you'd define the css per component and supply the CSS along with each component, separated into bundles with code splitting. With Hugo this is likely not an option. What you can do however is to split it per page type. You don't need the post css for the listing for example.

Separate the bundles in a way that does not duplicate, place shared rules in one bundle that is included for all pages and then page specific bundles that are also included for those pages.

## Critical CSS
First off, consider if splitting into parts isn't a better option. Critical CSS differs per page, window size and zoom/scale. It's really hard to get it right for all cases. It can be used as an automated way of splitting into parts (set size to render full page to get all CSS used in that page). Is it worth the complexity though?

Critical CSS is the CSS needed for initial render, anything that can be seen before scrolling or any other interaction. You can inspect it using Chrome's coverage tool (settings -> more tools -> coverage). For this page the first thought would be the footer, but some of the pages aren't long enough for the footer to always end up outside the view. Especially if desktop screens in portrait orientation are taken into consideration.

A common option for getting critical CSS right is to have it in mind for the design, avoiding the guesswork by trying to always show the same content first, you'll then also know that only that part needs to be included as critical css. Often this hides all content except the title or something similar though, which is likely a disservice to the user, who now has to interact with the page to see anything other than the title. It does buy you the time of however long it takes the user to finish reading the title and move on. But it also makes the TTI (time to interactive) metric more important.

Again, you probably shouldn't, but there are tools to generate it ([penthouse](https://github.com/pocketjoso/penthouse) is an alternative).

# Javascript optimizations
By default scripts will block rendering until they are downloaded, parsed and executed. Being aware of the alternatives allow you to move away from the defaults and reap some benefits for your users. Ideally, any scripts should be progressive enhancements. It can for example be used to fetch/render next page on likely click intent (hovering for a while).

## Render blocking
For advanced applications you'll need to differentiate which parts need to be blocking, sort of like with css above, you might have critical javascript. For a static site such as one generated with Hugo I have a hard time coming up with cases that needs to be render blocking though.

If you need to support really old browsers you can delay render blocking until as much content has been rendered as possible by placing your scripts last in the body tag. Nowadays though all you need to do is add an attribute to your script tags and make sure there are no inline scripts.

There are two attributes to choose from: `defer` or `async`. If you need to preserve the order in which the scripts execute or need them to execute after DOM ready and before DOMContentLoaded you should use `defer`. If the script is completely independent, where neither which order they execute in or when that happens matters (other than last), you should pick `async`. You likely built a bundle which should mean execution order isn't a factor and it shouldn't be messing with your content, so here's an example with async:

```go-html-template
<head>
  {{- $jsServerOptions := dict "minify" false }}
  {{- $jsBuildOptions := dict "minify" true "target" "es2015" }}
  {{- $jsOptions := cond (.Site.IsServer) $jsServerOptions $jsBuildOptions }}
  {{- $js := resources.Get "js/index.js" | js.Build $jsOptions }}
  <script async type="text/javascript" src="{{ $js.Permalink }}"></script>
</head>
```

This example includes conditionally minifying it based on whether it's a build or running in watch mode. It's hard to debug minified js, for css it doesn't matter because you're pretty much never looking at the rendered stylesheet anyway. Always minifying and including sourcemaps is a better approach since it increases the chances of the minified code to have been tested.

You should also include a hash of the content in the filename so that you can set more aggressive cache rules. It can be accomplished like this:

```go-html-template
{{- $jsWithHash := $js | resources.Fingerprint "sha512" -}}
```

Fingerprint can also be used to configure SRI (subresource integrity) using the integrity attribute of the script tag. It's not likely to help much for local resources (if they can replace that they can replace the html file with the integrity check too), but its important for external resources.

```go-html-template
<script integrity="{{ $jsWithHash.Data.Integrity }}"></script>
```

## Fetch/render earlier
The general idea is to prepare upcoming content beforehand so that the user spends less time waiting for it. You can do this by configuring prefetch/prerender hints, a really naive approach would be to just add prefetch hints to all linked content in lists for example. This would be quite wasteful though as the user is likely to pick one and proceed from there.

Gatsby had a feature where it would prefetch on hover, which I wanted to have here too. It was a bit excessive with downloads as it did it right away, I wanted the click to seem a bit more likely before prefetching. I also wanted to try prerender which will only be done for one resource, so there it needed to be even more likely before firing it.

Basing the likelihood of a click on how long the user would hover over the link before prefetching/prerendering seemed like a good option.

### Click intent
This is pretty straightforward, start timeout for executing the action on mouseenter, cancel on mouseleave and fire the action if the user kept hovering for the set amount of time. Disable after first execution.

```js
function onClickIntent(element, hoverTime, callback) {
  let isCancelled, timer;

  function disable() {
    element.removeEventListener('mouseenter', start);
    element.removeEventListener('mouseleave', cancel);
  }

  function fire() {
    if (!isCancelled) {
      callback(element);
      disable();
    }
  }

  function cancel() {
    clearTimeout(timer);
    timer = null;
  }

  function start() {
    isCancelled = false;
    timer = setTimeout(fire, hoverTime);
  }

  element.addEventListener('mouseenter', start);
  element.addEventListener('mouseleave', cancel);
}
```

### Create prefetch hints
Hints are configured using link tags in head, like this: `<link rel="prefetch" href="/some/path" />`. A page can have multiple prefetch hints and one prerender hint.

This part just hooks up a create hint method with the click intent. If the user hovers a link for 150ms, create a prefetch hint for that resource.

```js
const existingHints = [];

function createHintHooks(element) {
  let hint;

  function createPrefetchHint() {
    const href = element.getAttribute('href')
    if (!existingHints.includes(href)) {
      hint = document.createElement('link');
      hint.setAttribute('rel', 'prefetch');
      hint.setAttribute('href', href);
      document.head.appendChild(hint);
      existingHints.push(href);
    }
  }

  onClickIntent(element, 150, createPrefetchHint);
}

```

The events can then be hooked up by finding all links and executing the create hints method for them like this:

```js
document.querySelectorAll('a').forEach(createHintHooks);
```

### Including prerender
For the prerender hint, replace the rel attribute on the existing hint.

```js
function onClickIntent(element, hoverTime, callback) {
  let isCancelled, timer;

  function disable() {
    element.removeEventListener('mouseenter', start);
    element.removeEventListener('mouseleave', cancel);
  }

  function fire() {
    if (!isCancelled) {
      callback(element);
      disable();
    }
  }

  function cancel() {
    clearTimeout(timer);
    timer = null;
  }

  function start() {
    isCancelled = false;
    timer = setTimeout(fire, hoverTime);
  }

  element.addEventListener('mouseenter', start);
  element.addEventListener('mouseleave', cancel);
}

let hasPrerenderHint = false;
const existingHints = [];

function createHintHooks(element) {
  let hint;

  function createPrefetchHint() {
    let href = element.getAttribute('href')
    if (!existingHints.includes(href)) {
      hint = document.createElement('link');
      hint.setAttribute('rel', 'prefetch');
      hint.setAttribute('href', href);
      document.head.appendChild(hint);
      existingHints.push(href);
    }
  }

  function createPrerenderHint() {
    if (!hasPrerenderHint) {
      hint.setAttribute('rel', 'prerender');
      hasPrerenderHint = true;
    }
  }

  const isInternalLink = element.href.includes(window.location.host);

  if (isInternalLink) {
    onClickIntent(element, 150, createPrefetchHint);
    onClickIntent(element, 1000, createPrerenderHint);
  }
}

document.querySelectorAll('a').forEach(createHintHooks);
```

### Conclusions
According to caniuse prefetch should work for most browsers and prerender support is a bit worse. Firefox on linux (at least I don't think I've changed the setting) appear to default to the feature being turned off. I don't think any of the browsers actually prerender, but they do prefetch both the page and it's resources.

Taking this a step further we could actually just fetch the page, place it outside the view and then move it into the view, remove the old content and update the location on click. Could be very lightweight and would bring most of the good parts of a SPA (single page application), with pretty much none of it's bad parts.

## Get rid of bloat
This is kind of obvious, but get rid of stuff you don't need. It was also my main frustration with my previous setup using Gatsby, 340 KB of javascript to be downloaded, parsed and executed. Of which the only feature to have any value is fetching links on hover (and it does so rather naively too). You can read about the previous setup in [Getting a blog the hard way](/blog/getting-a-blog-the-hard-way)

## Javascript libraries
For a simple static site you probably don't need any libraries. Probably no javascript at all even. It's not a stretch to call the prefetch script above silly. Hugo doesn't seem like a great platform for an advanced application, consider switching to some other setup if that's what you're doing.

Any time you're thinking of adding a dependency, pause and consider. What would it do for you? What's the cost? How long would a specialized version take?

Cost is the hard part, that new dependency is an attack vector, what could a malicious actor do with it? what's the risk of that happening? what's the size of it compared to doing it yourself? Should you review it every time it's updated? How isolated is that dependency from your other code? How much work would it be to replace it? What's the maintenance cost compared to maintaining your own solution?

The cost in terms of time taken to write it yourself is apparent, many other factors are often overlooked. People skipping all of them is the only answer I can think of as to why all these 1-10 loc (lines of code) packages exist. A more selfish reason is that it's better for your career (at least long-term) to dig in and learn. Don't know how to do an is-even (yes, there's an npm package for that.. and it depends on the is-odd package) check? Fine, search for it, look for solutions, understand them and then implement it or consider the rest of the implications and pull it in, now that version of the dependency is reviewed at least (if you trust that what you looked at is actually what you'll be installing).

Be mindful of your dependencies.

### History
In 2016 there was a minor crisis because leftpad (padding left side of strings) was taken off of the package registry. Prime example of a library nobody should've installed, not because of the crisis, but because of the problem it solved. But it had millions of downloads and broke thousands of projects.

In 2021 we had ua-parser-js, which if you actually need much of the information it figures out is something that would actually save you countless hours. I don't like the fact that people do, but I get why it exists. Malicious versions of it were published to the npm registry and stole keys, cookies and passwords from anyone who installed it. We often have code review processes in place for any changes done by collegues. But we blindly trust random people on the internet because it's in a package? I think we "choose" not to because it's overwhelming.
