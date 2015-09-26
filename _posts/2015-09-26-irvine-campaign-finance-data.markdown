---
title: Liberating Irvine campaign finance data
layout: post
status: publish
---

Comrades of the Data Liberation Front rejoice! I come to report a successful action against the [campaign finance records of the City of Irvine](http://nf4.netfile.com/pub2/Default.aspx?aid=COI).

There is a shocking amount of money in Irvine politics. I received, without exaggeration, several pounds of direct mail in the run up to the November 2014 election, and I have very few guesses about who paid for it all. Irvine politicians are required to report donations, loans, and campaign expenditures. Unfortunately and unaccountably, they have not been required to file electronically. Several candidates have, in the past, chosen to file on paper; those filings are *not* data-entered by the [City Clerk's office](http://www.cityofirvine.org/city-clerk), and are posted online [as useless PDF page images](https://dl.dropboxusercontent.com/u/3720/temp/this_is_barely_disclosure.pdf).

Since the City Clerk didn't do their job, it's up to us. I contracted a freelancer to enter the data from the November 2014 election. It took her about 30 hours. [I have posted those records on Github](https://github.com/tdsmith/irvinecampaignfinance/tree/data). I assert that copyright cannot be held on this data and that these records exist in the public domain. I haven't done any systematic quality control but the spot checks I've done look reasonable. Counterintuitively, donor and vendor mailing addresses are redacted (presumably: by hand, laboriously) from the scanned PDFs, but not from the e-file data.

If you're passionate about public access and campaign transparency, I'd love to have your help cleaning up and analyzing these records; please get in touch! There's a lot of work left to do: these should be merged with the e-file data into a single coherent data set, and then it'll be time to try and generate some analyses of the data. It would be exciting to come up with some newsworthy pitches for one of the woefully under-resourced Orange County news organizations.
