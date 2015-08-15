---
layout: post
status: publish
published: true
title: Z-projections with Numpy
author_login: tim
author_email: tim@tim-smith.us
wordpress_id: 62
wordpress_url: http://blog.tim-smith.us/?p=62
date: !binary |-
  MjAxMy0wOS0xNiAxNzoxNzo1NyAtMDcwMA==
date_gmt: !binary |-
  MjAxMy0wOS0xNyAwMDoxNzo1NyAtMDcwMA==
categories:
- my research
- programming
tags: []
comments: []
---
<p>An imaging task I'm working on requires me to perform maximum intensity projections on multichannel z-stacks. The images are stored in xycz order in a multi-page TIFF container. The two channels are labeled CARS and SRS; in all stacks, I want to throw out the last CARS image but not the last SRS image and then write out the z-projection for each channel to disk. Let's see what this looks like!</p>
<p>https://gist.github.com/tdsmith/6588350</p>
<p>I'm using <a href="http://www.lfd.uci.edu/~gohlke/code/tifffile.py.html">tifffile.py</a> to open the TIFF container and read out the image data into a 3D numpy array, doing a little bit of fancy indexing to separate the channels. Then, I'm (expensively but concisely) performing the maximum intensity projection by sorting each channel along the Z axis. My images are only 512x512, so I can afford to read the entire stack into memory at once and sorting is pretty quick. Thought this was a cute method; wondered if there's a better way.</p>
