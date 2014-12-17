---
layout: article
title: A Robot for Scanning Rooms in 3D
tags: [Electronics]
excerpt: "At Floored Inc., I built a robot that takes 3D scans of interior
spaces so that they can be explored in the browser"
image:
  feature:  point_cloud_feature.gif
  teaser: point_cloud_teaser.gif
---



At [Floored Inc](http://www.floored.com/), I built a device for taking 3D scans of interior spaces.  Using a Hokuyo laser rangefinder, a 2D color camera, and a computer mounted on a motor-controlled tripod, I was able to create some
nice point clouds (as above), linked with image data.  "Nelson," as we called
v0 of this scanner, was built in Node.js (albeit with many C bindings).  Node's asynchronous style was both
a blessing and a curse for this project; on the one hand, it made it easy to parallelize tasks that could
be performed independently of each other, but was a pain when work needed to be done sequentially.
I would love to share my code, but unfortunately it belongs to Floored! You can see a picture
of Nelson and me [here]({{site.url}}/about), though!
