---
title: "Switching blog engine to Hugo"
author: Kristoffer Lind
description: "Recap of my adventure in moving from Gatsby to Hugo, why I did it and some comparisons between them."
categories:
- Development
tags:
- hugo
date: "2021-11-20"
---

Will this blog be entirely meta? a blog about a blog? Who knows? :) A collection of notes and thoughts from switching from Gatsby to Hugo, why I did it and a bit of comparison.

# Why?
The fact that Gatsby included tons of javascript for pretty much no functionality bothered me and I wanted to try something else. Hugo seemed like a good choice. It's popular, supposedly really fast and seemed to do what I wanted it to and not much else, meaning it should be simple.

# Getting started
- Get Hugo in whichever way you prefer.
- Create barebones blog using: `hugo new site blog`.

Next up is picking a theme. I wanted something minimalistic that looked decent (just like last time, I'm fine with changing logic, but I try to stay away from design work). After looking at the list of themes for a bit I picked tale. Originally built for Jekyll by Chester How and then ported to Hugo by Emiel Hollander.

I thought that it was likely that I'd be doing some changes, so I just forked it right away. Either that was a good choice, or I should've spent some more time picking as I've now spent a few hours making changes.

While inside the folder of your blog include it by adding it as a submodule `git submodule add ./themes/<name of theme> <git-url of theme>`. Set theme in config.toml `theme = <theme-name>` (theme name needs to match directory name in themes/).

# Add content
Start server to see the effect of changes `hugo server --buildDrafts`, I then copied the markdown files from Gatsby content directory to Hugo content directory, with a few minor changes to frontmatter. For this theme, posts in content/blog are posts and in content/ will be pages (this is where I put the about.md page).

# Deploy
Build minified with `hugo --minify` and push the ./public directory to gh-pages. Hugo recommends a 3rd party github action for deploying to gh-pages, but it seemed silly to pull in a dependency to replace a couple of commands.

```sh
cd ./public

# create repository
git init .

# git user needs to be configured
git config user.name "Deployer"
git config user.email "deploy-bot@github"

# setup remote
git remote add deploy-repo $REPOSITORY_URL
git checkout -b gh-pages
git add -A
git commit -m "Deploy"

# need to force push as it's not in sync
git push deploy-repo --force
```
REPOSITORY_URL needs to include the token like this: `https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/user/project`.

## Custom DNS
You'll need a CNAME file in the root of your $USERNAME.github.io repository. It can be added in the deploy script like this: `echo "$BASE_URL" > CNAME`. You'll also need to setup a CNAME record that points to $USERNAME.github.io.

# Conclusions (hugo vs gatsby)
Mostly related to the theme I picked, but I did spend quite a bit more time fixing the theme this time around. If you're better than me at picking a theme though it's easier to get started with.

Installation is just a binary, whereas with gatsby you'll need nodejs as a prerequisite followed by a gazillion npm packages. Theme modifications are also a bit simpler, with hugo it's mostly vanilla html and js, along with a templating language (and some golang if  you delve deeper). With gatsby you'll need to understand js, graphql, react and their ecosystems (very popular choices so lot's of people know them anyway, but still). Gatsby includes most of what you need for building a rather advanced application, if I were to do so I'd probably still want it to be built separately from the blog though.

You'll have more options with gatsby, but those also increase complexity. Gatsby does however come with more performance optimizations out of the box, with Hugo prefetching/prerendering related pages needs to be implemented by the theme or you. On the other hand Hugo doesn't require you to ship the entire js ecosystem along with your generated pages.

Gatsby:
- more options
- more built-in performance optimizations
- allows more aggressive caching

Hugo:
- less cruft included in generated page
- faster builds
- less complicated

# Interested in this particular setup?
Both the [theme](https://github.com/kristofferlind/tale-hugo) and [this blog](https://github.com/kristofferlind/kristofferlind.github.io) are available on github. I made a setup where the top repo is private and houses draft content and notes. That way I can include draft content when running locally using `hugo server --contentDir ../draft-content`. It also enables saving progress of those posts, while only exposing published content on github.

Theme modifications:
- remove disqus and google analytics support
- fix syntax highlighting and fit code blocks on mobile
- support menu, tags, categories and differentiating between pages and posts
- prevent FOUC (flash of unstyled content)
- prefetch/prerender links on likely click intent
- add document meta description
- rel canonical tags
- explicit width and height for images
- image processing (scale down to 800px width)
- remove custom fonts
- only display pagination if enough entries
- noopener on external links
- allow links to headers
- dark mode

Removing disqus and google analytics support might seem a bit odd, but I haven't tested them and I don't want to encourage their use. Custom fonts felt a bit silly when they were by far the heaviest part of page load. Image optimizations and prefetch/prerender are really the only interesting parts, you can read about them [here](/blog/hugo-theme-performance-optimizations).
