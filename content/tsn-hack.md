+++
title = "Supicious Subdomain Takeover Investigation"
date = 2019-12-26
draft = false
type = "post"

[taxonomies]
tags = ["SEO", "pwned", "encryption"]
+++

I stumbled upon strange search engine results leading to pages on a large corporate subdomain. These pages load
obfuscated and encrypted code from a myriad of domains, and it went un-noticed for months. By the time I got around to
finally writing this, I went and checked and it had been fixed.

<!-- more -->

Long story short: TSN, a Canadian sports broadcaster owned by Bell, had set up an app on Azure called "Check In" with
the domain `http://check-in.tsn.ca`, likely for some service that would allow you to tweet that you were at a live
event. In any case, this app was eventually shut down, but the developers forgot to change their DNS records pointing
the subdomain to some azure IP. When you take down an Azure app, anyone else can claim that ip, and hence your
subdomain! There is lots of prior discussion of this online, the attack is called a Subdomain Takeover. In this case it
looks like it was only used for SEO spam, but if TSN's main site had a XSS vulnerability, their whole system could have
been compromised!
