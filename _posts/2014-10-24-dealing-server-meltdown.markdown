---
layout: post
title:  "Dealing with a server meltdown"
date:   2014-10-24 16:39:00 +0000
categories: admin
---
Unfortunately I've recently suffered a major setback in that my previous server host has lost access to the servers they used from their provider. They were not a small company and shows that bad things happen to everyone. I am now using DigitalOcean for this website, thanks to the [Github Student Developer Pack](https://education.github.com/pack).

Despite previously saying I was going to be more vigilant with backups, this has since lapsed and as such, I have lost some data unfortunately. I aim to recover what I can as this site was cached by Google and others. I still need to setup the https certificates again and enable SPDY, my priority was getting this back.

What have I learnt? Well firstly that you can't trust anyone to keep your data 100% of the time, I'm now using DigitalOcean and I intend to use their snapshot feature and maybe try their backups.  It's also shown that I should pay more attention to how I setup applications the first time around as doing this again I fell into several traps that retrospectively, I remember previously.
