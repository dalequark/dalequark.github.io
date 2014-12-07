---
layout: article
title: "BeagleCache: A Network Accelerator for the Developing World"
tags: [Linux, Electronics, Networking]
excerpt: "A $45 web caching proxy designed to improve internet access in
developing world countries. 'Most Promising Research' at CMU's Information
Systems Conference"
image:
  teaser: beaglebones.gif
---

Internet access in developing world countries is both extremely slow and expensive, due largely to a lack of physical infrastructure.  As a result, web caching—wherein a central web cache stores data fetched by users in the network and provides subsequent requesters with a cached copy of that data—can be effective in decreasing the lag users experience in these regions.  Paired with another strategy called “bandwidth shifting,” in which an accelerator node located in a high-bandwidth region compresses data for and manages the freshness for accelerator nodes in low-bandwidth regions, caching can be more effective still.  I was curious to know if these sophisticated acceleration techniques could run on a cheap, single-board ARM computer, which could ultimately provide a simple plug-and-play platform for developing world network accelerators.  In my research, I attempted to create such a platform on a BeagleBone Black (BBB), a $45 Linux computer no larger than a credit card.  After testing the BBB's bandwidth in serving disk requests, I determined this cheap computer would have the chutzpa to serve as a web cache, and began developing caching/bandwidth-shifting software for it.  Using Node Javascript and C, I created a portable software stack that could run on the BBB, resulting in BeagleCache: A Low-Cost Caching Proxy for the Developing World.  
