---
layout: post
title: "Deploying tips for Web Applications"
date: "2012-02-27"
tags: 
  - "opinion"
  - "technology"
  - "web-development"
  - "webdev"
  - "deploy"
  - "hg"
---

I’ve seen the same error in too many projects. When we start a project normally we start with a paper, a white board or something similar. After the first drafts we start coding. In the early stage of the project we want to build a working prototype. It’s natural. It’s important to have a working prototype as fast as we can. The things are different in a browser. All works within a white board, but with a working alpha release we will feel “real” sensations.

Now the project is growing up. Maybe we need several weeks to going live yet. Maybe we haven’t even decide the hosting, but there is something we need to take into account even in the early stages of the project. **We need to build an automate deploy system**. The way we’re going to use to put our code in the production server. It’s mandatory to have an automated way to deploy our application. Deploy code in production must be something really trivial. Must be done with a few clicks. Hard to deploy means we are not agile, and that is not cool.

If the project is a "professional" one (someone pay/will pay for it), problems in the deploy means down times. Down times are not good. Our clients don’t pay us for those kind of problems. If the project is a personal project, a hard deploy system means that we’re going to be very lazy to improve our project. Deploy by hand is good idea only if we never forget anything and if we’re perfect. If not, it’s always better to have a build script.

It’s important to define different environments within our application. Modern frameworks such as symfony2 has a great way to define environments. It’s important to take into account that. Our code must be exactly the same in our development environment and at the production one. Exactly the same means exactly the same. If we need to change the code before deploy it into production server we've got a problem. A simple trick to define environments is create two ini files one with development data (database dsn, urls, paths) and another one to production. We can also use enviromnent variables, but keeping the source code identical.

So we need at least a build script to the source code, but we must remember that we also need to deploy database changes. Deploy database changes is a hard work, but source code can be trivial if we take into account a few details:

- Source code must be the same in all environments. Differences must be placed in configuration files.
- Never perform file-system operations directly with the console. We need to create scripts and execute the script to perform file-system operations. (folder creation, write-enables to log and caches, ...)

If we follow those simple rules we can create a very simple build scrip with our scm (git, mercurial).

The idea is very simple. One mercurial repository on development server. Another one on production server.

```ini
// .hg/hgrc
[paths]
prod = ssh://user@host//path/to/app
 
[hooks]
changegroup = hg update
```

Now we can easily clone the development repository. A simple _"hg push prod"_ will push code to the production server and [update](http://mercurial.selenic.com/wiki/FAQ#FAQ.2BAC8-CommonProblems.Any_way_to_.27hg_push.27_and_have_an_automatic_.27hg_update.27_on_the_remote_server.3F) the working repository. If you don’t have ssh access to the server maybe you need to build a custom script. Please do it. “Waste” your time creating your build script. It must works like a charm. Your life will be better. Another tools that will help us to build deploy scripts:

[http://capifony.org/](http://capifony.org/) [https://github.com/capistrano/capistrano](https://github.com/capistrano/capistrano) [http://www.phing.info/trac/](http://www.phing.info/trac/)

And that's all. Regards, [Gonzalo](https://twitter.com/#!/gonzalo123)
