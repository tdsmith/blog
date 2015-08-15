---
layout: post
status: publish
published: true
title: ! 'Getting mailto: URLs to work with Gmail in Fluid.app'
author_login: tim
author_email: tim@tim-smith.us
wordpress_id: 54
wordpress_url: http://blog.tim-smith.us/?p=54
date: !binary |-
  MjAxMy0wNC0wNSAwOTo1Mjo1NCAtMDcwMA==
date_gmt: !binary |-
  MjAxMy0wNC0wNSAxNjo1Mjo1NCAtMDcwMA==
categories:
- internet
tags: []
comments: []
---
<p>Because I couldn't find any documentation on this at all, anywhere, here is how to make mailto: URLs open in Gmail in <a href="http://fluidapp.com/">Fluid.app</a>.</p>
<ol>
<li><span style="line-height: 15px;">Open Mail.app. Open Preferences. On the "General" pane, set "Default email reader" to Gmail-Fluid.app.</span></li>
<li>In your Gmail Fluid.app instance, open Preferences. Open the "URL Handlers" pane.</li>
<li>Add a new handler by clicking the + button in the bottom left. Set URL scheme to mailto and set URL replacement to:<br />
https://mail.google.com/mail/?extsrc=mailto&amp;url=mailto:<br />
Make sure to include the final colon.</li><br />
</ol><br />
Mailto links should start working immediately, so when you click on an email link on a website, it will open a compose window in your Fluid.app instance. This also provides a hint about how to use the URL handler feature in Fluid.app in general, though I haven't tried anything else.</p>
<p><strong>Update:</strong> This changed in 1.7, I think, without any mention in the Changelog. Instead of following step 3, add a new pattern set in the URL Handler preference dialog. Change the pattern to match&nbsp;mailto:* and change the Javascript block to read:</p>
<p><code>function transform(inURLString) {<br />
return 'https://mail.google.com/mail/?extsrc=mailto&amp;url=' + inURLString;<br />
}<br />
</code></p>
<p>And you're set!</p>
