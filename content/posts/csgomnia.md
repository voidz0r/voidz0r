---
title: "A phishing attempt on Steam that became a Qrljacking research"
slug: "a-phishing-attempt-on-steam-that-became-qrljacking"
date: 2023-08-11T13:59:55+02:00
weight: 1
images: ["logo/void.png"]
---

# Preface
This is a brief story about a phishing attempt made by a user that turned out into a little research.

# The initial contact
The first contact I had was with a profile named Abigail🌻 who sent me a friend invite on steam.

![First contact with the user](/csgomnia/friend_request.png)

After a few minutes I received a message from the user, asking for a vote on a specific CS:GO AK-47 skin. Unfortunately I didn't take screenshot of the conversation because I got blocked and the messages are gone.

The user asked to visit a specific website called `csgomnia` at `https:[/][/]twitch[.]csgomnia[.]tv[/]`, which presented the following page:

![The website](/csgomnia/the_website.png)


# The website

At a first sight the login process is more than legit:

![The login window](/csgomnia/login_window.png)

From the above:

- The domain is correct
- SSL Certificate is trusted
- The page is correct
- The links on the page point on the `steamcommunity.com` website
- The look and feel is identical to the original
- There are no typos, glitches or spacing issues whatsoever.

An uncaught eye could just blindly trust the website right? ..Right? 

![Chrome Console](/csgomnia/chrome_console.png)

With some astonishment, I've notice that every block of the website is carefully built using CSS/images. Including the header bar, the minimize/maximize/close buttons etc. A very well done job. But what happens if someone inserts a correct login/password?

Right after the login, I noticed that the form always returns a "wrong username/password" combination, so the website cannot verify if credentials are valid. Speaking with "abigail", she/he said "It happened to me as well. Use the QR code instead?".

After making sure that I had OTP in place I've inserted my credentials and this happened:

![Login attempt](/csgomnia/login_attempt.png)

Obviously I changed the password afterwards

# The QR code

While the login/password combination is just plain old phishing technique that uses Javascript to build the popup, the QR code in the phishing page is actually pretty interesting since it uses some sort of QR generation URL from steam.

Differently from the official steam platform that generates a QR code every N seconds, the phishing page isn't able to do the same thing but, apparently, uses some sort of QRChallengeURL that is sent over HTTP on the `/ccelhsuqeiaycnvnk` endpoint of the `samiphonepack[dot]com` domain.

![QR Code Challenge URL](/csgomnia/qrcode_logics.png)

# Digging into the QR code logics

A deeper analysis on the QR code authentication flow showed some interesting aspects. The QR code itself isn't either encoded nor encrypted, but contains the same URL showed in the screenshot above like `https://s.team/q/1/8037143460412323909`

The URL points to a steam page and nothing happens if the URL is visited, though the URL is used in the authentication workflow. The first request hits the `/IAuthenticationService/BeginAuthSessionViaQR/v1` on the `api.steampowered.com` host.

![first request of the auth workflow](/csgomnia/auth_first_request.png)

While the response is in Protobuf format, it is somehow difficult to reconstruct the object without a schema. There is a project called `blackboxprotobuf` from NCC group which supposedly converts every protobuf encoded message into a list-like structure, but I had no luck trying to decode/re-encode the requests I had. So I decided to go manually.

So until now we know that the response is protobuf encoded and that will be used in the next requests. Right after generating this string, the website sends several requests to the `/IAuthenticationService/PollAuthSessionStatus/v1` endpoint to check if the user actually scanned the QR code with the Steam Mobile App (Steam Guard).

![Second request](/csgomnia/auth_second_request.png)

By comparing the bytes of the first and the second request, we can see some patterns:

![First request](/csgomnia/auth_request_compare_1.png)
![Second request](/csgomnia/auth_request_compare_2.png)

Subsequentially the application regenerates a new QR whenever the old one has expired.

## Dissecting the QR code authentication workflow

By now we know that:
- The QR code is in clear-text format and contains an URL
- The `/IAuthenticationService/BeginAuthSessionViaQR/v1` endpoint initiates the auth process. The returned object is constructed as:
    - First 11 or 12 bytes of the response. The size of it depends on some unknown factors.
    - From 12th or 13th byte onwards the QR code URL. The URL changes at regular intervals and it's returned by the `/IAuthenticationService/PollAuthSessionStatus/v1` endpoint
    - 17 bytes from the 53rd position is the last part of what it looks like a signature.
    - The rest of the code is ignored
- Once the user approves the QR code, a JWT token is returned from the previous URL
- The JWT token is then sent to the `https://login.steampowered.com/jwt/finalizelogin` along with:
    - The sessionid, which is set as a cookie on the home endpoint `/`
    - The redir parameter, which is most probably the callback URL
- The `finalizelogin` endpoint returns the most important parameter in order to be authenticated, the `steamLoginSecure`, which is passed to the `https://steamcommunity.com/login/settoken` endpoint, along with the sessionid


## Limitations
Due to a protection mechanism that denies the access whenever a parameter is different between requester and approver, whether IP or country or device, the attack surface is pretty limited. The attacker needs to know exactly the location of the target account and try to contact the Steam endpoints by using the same country or ASN.

## Risks

The risks, like what could have happened to myself, is a total account compromise where the attacker impersonates the user on the Steam platform.


## Remediation advices

The vulnerability is caused by a lack of encryption for the QR code. Since a user could easily generate one of them, it's possible for an attacker to forge the them and attempt a phishing attack against uncaught users.
Unfortunately there is no particular way to mitigate this issue in a QR-based authentication scenario. The general advice would be to avoid the mechanism completely and simply rely on user-password/OTP flow.
Fortunately enough, Steam actually checks if the requester comes from unusual and unseen locations. Trusted locations are mainly based on previous successful login attempts and e-mail verifications.

In short, never scan any QR code from a website different than the Steam one.


## Video

{{< video "/csgomnia/poc.mp4" "my-5" >}}

## The PoC

The PoC code will be published once I have a confirmation from the vendor (Valve).