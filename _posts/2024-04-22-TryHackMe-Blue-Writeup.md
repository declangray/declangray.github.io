---
layout: post
title:  "TryHackMe Blue Write-up"
date:   2024-04-22 +0800
categories: write-up
---
This write-up will be walking through the [Blue](https://tryhackme.com/r/room/blue) machine on TryHackMe. This is an easy machine and TryHackMe says it should take around 30 minutes to complete. We also know that this is a Windows machine with some common misconfiguration issues that we may be able to leverage to gain access.

**Question 1** Scan the machine.
This first question is quite simple, we just need to do a port-scan using Nmap. We can do a basic Nmap scan just simply using the command:

{% highlight bash %}
nmap [MACHINE IP ADDRESS]
{% endhighlight %}

And we get the result:


