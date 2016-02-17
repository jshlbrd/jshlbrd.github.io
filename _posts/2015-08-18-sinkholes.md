---
layout: post
title: Digging up sinkholes
categories:
- blog
---

Detecting sinkhole server requests is something I have never been fully satisfied with, and [this thread](http://t46791.security-ids-snort-emerging-sigs.securityupdate.info/et-trojan-dns-reply-sinkhole-t46791.html) from 2013 sums up my dissatisfaction. IDS rules that alert on sinkhole servers don't provide you with enough data to make a decision on what to do next.

These IDS rules create two problems:

- You (probably) don't know what endpoint made the request
- You don't know what domain was requested

This post doesn't address the first problem. You can't get around the fact that you need some information to connect the DNS request to an endpoint. What this post addresses is the second problem.

## The problem with detecting sinkhole servers

The problems listed above are best explained through a couple scenarios.

Scenario A:

- You run an IDS on the edge of the network
- You do not collect PCAP
- You see this alert: "ET TROJAN DNS Reply Sinkhole - Anubis - 195.22.26.192/26"

This is the worst case scenario. The data available to you from the IDS includes connection flow data (source IP + port, destination IP + port), timestamp for when the event occurred, and metadata surrounding the alert (action, category, severity, etc). Keep in mind that the source IP address is your external DNS server. Since you don't collect PCAP, you have no hope of capturing the domain that was requested.

What this alert tells you is that some endpoint in your environment requested a domain that resolved to the Anubis sinkhole. You don't know what domain the endpoint requested or what endpoint performed the request. While you might get lucky by spending time digging through DNS server logs and attempting to match up timestamps with the alert timestamp, in my opinion this alert is a dead-end.

Scenario B:

- You run an IDS on the edge of the network
- You do collect PCAP
- You see this alert: "ET TROJAN DNS Reply Sinkhole - Anubis - 195.22.26.192/26"

This is the best case scenario. You have all the data available to you from Scenario A, plus some or all of the raw packet data. You might think this is great, but, as you'll read, this still isn't ideal. Having PCAP of the session kind of solves the problem of knowing what domain was requested-- while the domain exists in PCAP, that data is not quickly accessible. You have to identify the specific session that the DNS activity occurred in (some tools can do this for you), then review the PCAP manually to identify the requested domain. Though this is the best case scenario, it still introduces a lot of unnecessary steps (identifying PCAP, retrieving PCAP, analyzing PCAP) into what should be a simple process.

Both of these scenarios relied on having additional data (DNS server logs, PCAP) to identify the domain. What if you could find the domain without needing additional data?

## What if you just instantly knew what domain was requested?

You can probably tell where this is going. With enough time and effort, the best case scenario will eventually lead you to the requested domain. Wouldn't it be better if you just had the the domain without having to do any extra work and without having to rely on multiple sources of data? I wrote [a Bro script](https://github.com/CrowdStrike/cs-bro/tree/master/bro-scripts/sinkholes) that does exactly that by building upon the dns.log.

The script uses the Bro Input framework to read a file that contains a list of sinkhole server IP addresses / netblocks. Whenever Bro sees a DNS reply that contains answers, it will reference the supplied list of sinkhole servers, check for the answer in the list, and append the boolean field "sinkhole" to dns.log. (By default, this field's value is false.) Below is an example dns.log that uses the Anubis sinkhole servers:

{% highlight bro %}
#fields	ts	uid	id.orig_h	id.orig_p	id.resp_h	id.resp_p	proto	trans_id	query	qclasqclass_name	qtype	qtype_name	rcode	rcode_name	AA	TC	RD	RA	Z	answers	TTLs	rejected	sinkhole
1359930253.708622	CHTzdSTuBRxDjyX81	172.16.253.129	53	4.2.2.2	53	udp	48752	www.ald-transports-express.eu	1	C_INTERNET	1	A	0	NOERROR	F	F	T	T	0	195.22.26.231	100.000000	F	T
{% endhighlight %}

By looking at the fields *query* and *sinkhole*, we immediately know that the request (www.ald-transports-express.eu) was answered by a sinkhole. (In this case, it's a Sality domain.)

That's it, it's that simple. If you run Bro, then you can grab the script, build a list of sinkhole servers, and start tracking sinkhole server requests with ease.
