---
layout: post
status: publish
published: true
title: PIL, Cairo, and numpy
author_login: tim
author_email: tim@tim-smith.us
wordpress_id: 134
wordpress_url: http://blog.tim-smith.us/?p=134
date: !binary |-
  MjAxNC0wOC0xMSAyMzowOTowOCAtMDcwMA==
date_gmt: !binary |-
  MjAxNC0wOC0xMiAwNjowOTowOCAtMDcwMA==
categories:
- Uncategorized
tags: []
comments: []
---
<p>Here are the incantations you need to perform to create a Cairo surface from a grayscale image in a Numpy array, doodle on it, and convert the doodled-upon image back into a Pillow image so you can save it as e.g. a JPEG.</p>
<p>{% gist 9591abe9dd7ebe280d82 %}</p>
