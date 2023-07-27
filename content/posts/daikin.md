---
title: "CVE-2018-20139 - Daikin Emura Series - Arbitrary Remote Control via DNS Rebinding"
slug: "cve-2018-2013-daikin-emura-series-arbitrary-remote-control-dns-rebinding"
date: 2018-12-14
---

There is a lot of hype around DNS rebinding vulnerability and vulnerable IoT devices, including home cameras, air conditioners or climate control devices; this flaw will cover the lack of origin checking on HTTP requests.

## DNS Rebinding Attack
DNS Rebinding allows an attacker who has control on a DNS server to communicate with a device connected to the internal network of the victim.

Normally, these kind of attacks are mitigated by the Same-Origin policy header, implemented by most of the browsers, that will block every attempt to load external content that does not belong to the original domain.

The idea of the attack is to configure an evil DNS server that will respond to a specific A query and will return two IPs for two consecutive requests; the first one will return the attacker's IP with a short TTL (1 sec), the second one will return the original IP of the device.

Since most browsers use different caching systems with different expiration times, a minimum amount of seconds is needed to perform the attack. In order to achieve a perfect exploitation scenario, the victim needs to stay on the attacker's website for at least 60 seconds (on Firefox, it may change with other browsers).

![Cache poisoning - request flow schema](/daikin/image-21.png)

## The attack
First, we configure the DNS server using FakeDns tool; we just need one entry for our attack scenario.

NOTE: In our PoC we used NAT addresses, an internet address can be used for the fake DNS server.
![Configuring DNS rebinding](/daikin/image-22.png)

## Configuring DNS rebinding
Now the evil DNS server will respond to A query for rebind-example.voidzone.it host, the first parameter before the '%' is the TTL.

Next step is to build an HTML page, entertaining our user with a youtube video and waiting for the timeout; when this occurs, an XMLHttpRequest is sent to the device.

![Hosted page](/daikin/image-23.png)

![Wait page](/daikin/image-24.png)
And, after 60 seconds we get the response from our Daikin device:

![Response from Daiking Emura Rest API](/daikin/image-25.png)


<iframe title="vimeo-player" src="https://player.vimeo.com/video/564623325?h=f4cec3f9a2" width="640" height="360" frameborder="0"    allowfullscreen></iframe>

​


## Fix Available
The vulnerability has been fixed in the version 3.3.6 of the firmware.

## Credits
I would like to thank Mr. Pieter-Jan Decherf and the Daikin Europe team for the quick responses and updates on the remediation measures implemented.

## Responsible disclosure time-line
- 25/07/2018 - First contact with Daikin Europe
- 26/07/2018 - Got response, asking for report
- 07/08/2018 - Report+PoC sent
- 08/08/2018 - Vulnerability confirmed
- 14/09/2018 - Vulnerability will be fixed by the end of October
- 31/10/2018 - Official update released (firmware 3.3.6)
- 19/11/2018 - Paper draft has been sent to Daikin Europe team
- 13/12/2018 - CVE-2018-20139 assigned