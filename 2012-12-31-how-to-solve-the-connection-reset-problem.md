---
title: 'How to solve the "Connection Reset" problem'
date: "2012-12-31 20:50"
comments: true
categories: Informatics
tags: ["Written In English","Black Technology",GFW]
---
考虑到安全原因，这篇文章用英语写成。如果你没有足够的勇气读完它，请自觉退出。

This article is written mainly for those people in China. Be sure you are enough familiar with what you are reading and what you try to do.

================================================================

As we all known, in China mainland, we cannot visit sites like YouTube Facebook Twitter. And the Google sites are out of service frequently. It's because the Chinese government used some technical methods to prevent us from visiting them. The government has setup a system to do this. It's called the [Great Firewall of China](https://en.wikipedia.org/wiki/Great_Firewall) (GFW). This system keeps look on the gateway export. And if it finds something unusual. It will stop the connection.

The system usually inject a RESET into the TCP connection. To prevent this, we can use HTTPS(The S means Secure) instead of HTTP. So the system can not inject the RESET any more. It's easy to perform. You just need to replace the "http://" part of a URL with "https://". And the URL will look like this "https://www.facebook.com". Most of the sites support a HTTPS connection.

Unfortunately, this will not always works. Because the system also used another method called "DNS Redirection". As we all known, the computers on the Internet are identified by IP address. But human can't remember them easily. So we use some meaningful phases called "Domain". Some computers on the Internet provide the kind of service to translate the domains to IP address which is the only form computers can recognize. They are called the "DNS Server". DNS Redirection is that the DNS servers won't return the correct IP address (usually were instructed to do so) so that we can't visit the particular sites.

Luckily, we can assign an IP to a domain manually. That's the function of a file called [hosts](https://en.wikipedia.org/wiki/Hosts_(file)). There's a project called [smarthosts](https://code.google.com/p/smarthosts/) on Google Code. It provided a set of IP address you may use. Paste them to your local hosts file and enjoy the Internet.

I will write more about the Internet censorship and how to avoid it. Check back later.
