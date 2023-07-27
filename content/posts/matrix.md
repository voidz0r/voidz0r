---
title: "Matrix Synapse 1.12.3 - SSRF and Cache poisoning"
slug: "matrix-synapse-ssrf-cache-poisoning"
date: 2020-01-14
draft: true
---

## tl;dr
The Matrix Synapse servers have been found affected by a security issue about the lack of a validation system for "Server-to-server" API leading to SSRF and Cache poisoning subsequently marked by the team as “feature” or “intended”.  In short, a malicious user, if not specifically denied by configuration files, could effectively load malicious content using what is called the “Server-To-Server” API and, since the caching mechanism allows a lifetime of 24h, it could allow to host arbitrary files for that duration.  After a discussion with the team they came to conclusion that this is an intended behavior and the best solution is to use an offload antivirus to scan for common malicious patterns on the uploaded files.

I want to thank the security response team for the response and for the brief discussion.

## Responsible disclosure timeline
​
| Date | Description |
| ---- | ----------- |
| 2020-04-29 | Vendor is notified |
| 2020-05-31 | First contact with the security response team |
| 2020-06-08 |	Discussion about the issue, marked as "intended" |
| 2020-06-16 | Disclosure |
​

## Introduction
Matrix is an open standard for a decentralized, interoperable and real time communication over IP.  This standard is widely documented and it defines a set of open APIs for communications. Its main point of strength is the ability to have a decentralized, securely encrypted and segregated federation of servers, giving the opportunity to users to take advantage of the following capabilities:

- Instant Messaging (IM)
- Voice over IP (VoIP)
- Internet of Things (IoT) communication

A good amount of integrations (or bridges) with external IM/VoIP protocols like:

- IRC
- Skype
- Telegram
- WhatsApp
- E-Mail
- Hangouts
- WeChat
- Twitter
- Keybase
- A good and simple set of JSON Rest APIs that allows developers to create their own clients or bots
- Fully opensource
- KISS principle
- End-To-End encryption feature by design

## The federation concept
The idea behind a federation server is simple and yet effective; an isolated, on-premise Matrix server will be capable to interact with other servers using a common API.  Following a schema taken from the matrix.org website:

![The matrix federated architecture](/matrix/image-8.png))

Every matrix client connected to a federated server can, if not specifically denied by configuration, communicate with other people from other servers or channels. Private keys will not be shared among this process.  The communication between servers is documented in the Federation API (or Server-Server API) docs.

## The federation API
Matrix home-servers use the Federation API in order to communicate with each other. Home-servers interact with these APIs in order to push messages to each user, join channels, retrieve history and query information about users using JSON format.  One part of this is called the “Server discovery” API and it will be the subject of this research.

## The bug - SSRF and Cache Poisoning

If not specified, matrix home-servers allows to integrate with other federated servers by using an exposed API. The following screenshot is an example of a file download that is located in the matrix.org federated server:

![original request](/matrix/image-9.png)
The highlighted part defines which federated server should be used to download the file. By giving a different URL, the following response is received:

![updated request](/matrix/image-10.png)

![callback received](/matrix/image-11.png)
So, the federated server is looking for a `server` file inside the `.well-known/matrix` path. As specified in the documentation, the server is expecting a JSON formatted file that specifies the information about the federated server like the following:

![expected config file](/matrix/image-12.png)
With this in mind, some aspects should be considered:

- Loopback or localhost address, even specified as numeric IPs (like 127.0.0.1) are blacklisted by default (the “federation_ip_range_blacklist” parameter on “homeserver.yaml” file)
- Using protocols different from HTTP or HTTPS will result in a timeout
30x redirects are accepted
- The cache lifetime is statically set to 86400 seconds (1 day). When a subsequent request to the same host is made, the result will be the same as the last one (this opens a Cache Poisoning scenario).
- If not specified, the 8448 port will be used
- The TLS certificate must be valid, so no unencrypted protocols can be used (if not allowed from configuration file)

The next step is to create a configuration file that points to the attacker’s-controlled endpoint:

![new configuration file](/matrix/image-13.png)
And, after issuing another request, a callback is received:

![new request](/matrix/image-14.png)

![callback](/matrix/image-15.png)
In order to obtain a full exploitation scenario, a “malicious file” is created.


![creating a malicious file](/matrix/image-16.png)
And a new host is configured on .well-known/matrix/server file:

![new matrix server configuration file](/matrix/image-17.png)
Then, a new request for the “malicious_file” is made:

![malicious file is cached for 24h](/matrix/image-18.png)

So, from now on, this file will be available in matrix.org servers for 24 hours; even if the external federated server is not available.

![file download](/matrix/image-19.png)

## Impact of this vulnerability
An attacker could exploit these two vulnerabilities in order to:

- Host malwares or, generally speaking, infected files
- Use it as a C2 server in order to broadcast files or messages
- Use it to launch denial of service attacks
- Hide the attacker IP while sending HTTP requests (using the server as a reverse proxy).
- If an XSS vulnerable application is hosted on the same host, this could bypass the Same-Origin Policy and allows an attacker to embed
JavaScript files directly into the application, as long as share application cookies.

## Affected versions
At the time of writing, the affected version is `1.12.3` as reported from the matrix.org API:

Remediation
The remediation is simple and it’s just a matter of specifying a list of whitelisted federated servers on the “/etc/matrix-synapse/homeserver.yaml” file. Unfortunately, this parameter is not set by default nor mandatory while installing the matrix server and there are some considerations to take in mind while applying this remediation (take a look at “Final considerations and thoughts” chapter).


![remediation](/matrix/image-20.png)

## Final considerations and thoughts
There are some aspects to take in consideration while analyzing this vulnerability. This flaw is basically leveraging on a feature and, in some cases, a remediation like the one proposed in the “Remediation” chapter isn’t feasible.  A server like Matrix.org that SHOULD allow every federated server to connect and interact; the whitelist scenario may not be applicable.

Keeping the default settings and letting every federated server to inter-communicate with each other is a serious security risk. This, at the same time, would effectively help the “Federated network” concept to grow.

Fixing this vulnerability could be a serious hit on the matrix-synapse federated network’s idea. If the server blocks the ability to download files or perform requests to external locations, shared files will not be share among other federated servers. The result is that User A on server 1 cannot share files with User B on server 2, and vice-versa.

Lastly, while the note on the configuration file “/etc/matrix-synapse/homeserver.yaml” states:

*N.B. we recommend also firewalling your federation listener to limit  # inbound federation traffic as early as possible, rather than relying  # purely on this application-layer restriction. If not specified, the  # default is to whitelist everything.*

This clearly isn’t a good solution if the server needs to be inter-connected with the rest of the world.