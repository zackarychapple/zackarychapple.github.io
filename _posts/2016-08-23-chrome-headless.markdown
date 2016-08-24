---
layout: post
title:  "Chrome Headless build"
date:   2016-08-23 20:40:56 -0500
categories: chrome
---
## Intro
Back in June Sam Saccone had a post about Chrome Headless that seemingly was loved by many with a plethora of likes and re-tweets.
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Headless Chrome is coming so soon 🎪!!! Say goodbye to the myriad of hacks to run Chrome and phantom on CI. 🌊<a href="https://t.co/ucDBbgpBO0">https://t.co/ucDBbgpBO0</a></p>&mdash; Sam Saccone (@samccone) <a href="https://twitter.com/samccone/status/739166801427210240">June 4, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

With Chrome Headless there are a multitude of opportunities for testing but my goals were a little different.  I wanted to create a way to capture screenshots and generate PDF's.

## Building Chrome headless
1. Create an Ubuntu EC2 on AWS
There are
![EC2]({{ site.url }}/assets/headless-assets/EC2.png)
![EC2 Type]({{ site.url }}/assets/headless-assets/EC2Type.png)
![Storage]({{ site.url }}/assets/headless-assets/Storage.png)
![EditedStorage]({{ site.url }}/assets/headless-assets/EditedStorage.png)

