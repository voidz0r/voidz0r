---
title: "Facebook chat / dashboard content injection"
date: 2018-01-03
---

Some time ago a friend of mine was spear-phished with a message through the Facebook chat, this happened before Facebook patched the chat application, allowing to exchange of messages only between people connected as friends. While I was analyzing the message, I noticed that it looked like a legit message, with a link to a newspaper page, a title, body and image, but also with a pseudo-random generated domain like the killswitch domain used by WannaCry ransomware, written in a small gray text area below the description.

Crawling on Google I found out that someone already reported a similar finding to Facebook Security Team but it was not exactly the same issue that I discovered; they were just talking about the ability to to inject whatever URL inside opengraph tags. So I decided to go straight through that to figure out something more.

## Analysis
The Facebook chat and board features implement a functionality to generate a preview box whenever someone write a full URL in a message; the Facebook crawler, then, will scan the target website, take some markup headers parameters like og:title og:description along with an icon of the page and returns a preview box. This is a part of the already known OpenGraph Markup Language, nothing new.

What I didn't expect is that the Facebook web application uses client-side data to generate the final preview box.

![PoC](/facebook/0.jpg)

## Getting hands dirty
Starting Burp Suite I started to intercept all the traffic from and to the website; I used repubblica.it as a target, an italian newspaper.

**Note**: I tested it out by sending message to myself in order to not violate facebook disclosure rules.

![PoC](/facebook/image.png)
And, by changing those URLs:

![PoC](/facebook/image-1.png)
And after forwarding the message:

![PoC](/facebook/image-2.png)
The original URL, the icon, title and body is taken directly from the repubblica.it website, except for that little grayed text below "google.it". When we click on it we will get:

![PoC](/facebook/image-3.png)
Bullseye!

At this stage we know:

- The output URL is what we sent from AJAX POST request
- URLs are not encoded
- Server does not validate if the input URL is the same that we typed it.
- Server does not validate if the result mismatch with our tampered request.

## Mitmproxy to the rescue!
Mitmproxy is basically a python software that acts as an HTTP proxy like BurpSuite but with the ability to change every aspect of an HTTP request or response at runtime; it can even be used as an interactive console to change/inspect HTTP requests.

I wrote a small python script that takes everything I got from the POST request sent to /messaging/send endpoint and replaced every URL (www.repubblica.it) with our domain, using a regular expression.

Since the server somehow does not like query string parameters, I configured a website to perform a permanent redirect to a youtube video.

![mitm](/facebook/image-4.png)
![mitm](/facebook/image-5.png)

After clicking on it:

![poc](/facebook/image-6.png)

## Foot notes
Facebook security team is already aware of this issue and this is the response:
![response](/facebook/image-7.png)

So they will consider it as "low-impact risk" and, most probably, they will not fix it.