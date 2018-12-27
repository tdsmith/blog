---
title: "A minimal Gmail API example"
layout: "post"
---

Once upon a time, I created a whole lot of gmail labels that I don't use anymore.
I wanted to clean those up in anticipation of Inbox going away.
There are a lot of outdated examples for creating a Google API client application,
so I wanted to capture the form of the "modern" solution I found.

The solution is captured [in a notebook](https://gist.github.com/tdsmith/ca5766468827280481dcb7ae4e62f876).
It uses the `google_auth_oauthlib` library to create an authorized `requests` session, and then just does REST-y things
instead of using a client library wrapper.

To use it, run the first cell, visit the link, copy-paste
the authorization code into the text widget,
and then run the rest of the notebook.
