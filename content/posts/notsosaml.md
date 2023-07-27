---
title: "SA-CONTRIB-2021-036 - NotSoSAML - Privilege escalation via XML Signature Wrapping on Miniorange Drupal plugin"
slug: "sa-contrib-2021-036-notsosaml-miniorange-privilege-escalation-via-xml-signature-wrapping"
date: 2021-07-09
---

This is a brief story about how a vulnerability on a drupal plugin has been found, which, when not configured correctly, could allow an authenticated user to perform a privilege escalation attack on the Drupal platform. This plugin, as stated in the usage section of the drupal website, is used by roughly 522 websites in the current month (June 2021) - ranging from version 7.x to 8.x Plugin stats.

## Some history and background
SAML is a recognized and widely adopted security standard used by web applications for authenticating and authorizing users. The markup language, as someone could guess, is XML and the authentication payload is base64 encoded before being sent over HTTP POST request as a body parameter. The whole point of the SAML is to provide a standardized mechanism to authenticate/authorize clients to multiple platforms and, at the same time, gives the ability to have a single point of entry for authentication flows, also known as Single Sign On.

## The authentication parties
In a SAML authentication workflow there are three parties involved:

- **The client**: this could be a browser or an application that needs to authenticate
- **The Identity Provider**: this entity handles the authentication process and returns a valid SAML message that will be submitted to the Relying Party.
- **The Relying Party**: this party receives the base64 encoded XML message, verify its signature and integrity and authenticates the client to the platform.

The entire workflow is also called Single Sign On.

The following schema summarizes the above points:
![SAML schema](/notsosaml/image-26.png))


## What are SAML assertions
SAML Assertions are claims about a subject. These claims should be verified once received by checking the `<Signature>` tag inside of the SAML body message.

![SAML signature](/notsosaml/image-27.png)

SAML is a ~20y old security implementation, nonetheless many research revealed that most of the frameworks do not implement this protocol in a correct way. The most common pitfalls are:

- **Difficult-to-understand signing algorithm**: contrary to simpler data formats such as PKCS#7 that produces a single hash for the entire document, the XML signature is quitely fragmented over the body of the message itself. With this confusion, it is quite common to forget some critical parts of the message.
- **The XML markup language itself**: Long, overly-complicated and not-so-human-readable.
- **Architecture of the SAML frameworks**: it is quite common to make one entity for the verification and the signature processes, while it is heavily recommended to split those two operations into different and, possibly, separated components in order to avoid confusion or bad application behaviors.

## Some of the most known attacks for SAML
Most common SAML implementation vulnerabilities can be summarized like this:

- **Message replay attacks**: The application doesn't check a specific unique-ID and allows an attacker to replay the same message multiple times in order to maintain or renew a valid authentication on the platform.
- **Invalid/missing signature**: The application doesn't check the provided signature value against the Certificate Authority, or the the current CA is self-signed. This could allow an attacker to replace the CA with a custom-made self signed one.
- **SAML Recipient attack**: The IdP doesn't check if the message comes from the intended service provider, allowing an attacker to send arbitrary requests to the identity provider.
- **XML Signature Wrapping (XSW)**: These well known methodologies exploit how the application checks for multiple assertions, multiple signatures or behave differently based on the situation. There are several known methodologies that can be used such as:

    - Applying a cloned unsigned copy of the Response before/after the existing signature
    - Applying a cloned unsigned copy of the Assertion before/after the existing assertions.
    - Changing a value in the signed copy of the Assertion block and add a copy of the original Assertion without the Signature.
    - Adding an Extensions block with a cloned unsigned Assertion
    - Adding an Object block containing a copy of the original assertion with the Signature removed.
    - XML eXternal Entity (XXE): Since a SAML message is a plain XML file, some XML parsers could accept XXE payloads that leverage on the SYSTEM functionality of the XML parser or load an external DTD file - also known as Out-Of-Bound XXE.

## Configuring the environment for exploitation
### Configuring SAML integration on Auth0 and Drupal
The Miniorange_saml plugin allows Drupal administrators to integrate with third party identity platforms using the SAML protocol.

For demonstration purposes, the plugin has been configured with the auth0 provider. The first step is to create an application in the auth0 dashboard and associate a SAML addon with it.

![auth0 config](/notsosaml/image-28.png)


Once created, those settings should be placed in the miniorange_saml plugin section.
![plugin config](/notsosaml/image-29.png)


In order to exploit this vulnerability, the plugin must be configured without the Either SAML reponse or SAML assertion must be signed option and without the x.509 certificate.

![bad config](/notsosaml/image-30.png)
Once everything is ready, the following option should appear on the drupal site.


![drupal new option](/notsosaml/image-31.png)

### Testing it out and exploiting the XSW
Clicking on the `authenticate with auth0` option should bring up the auth0 login page.

![authenticate with auth0](/notsosaml/image-32.png)

And, once successful, it will return to our website with our new user.

![login success](/notsosaml/image-33.png)

By using a proxy like Burp Suite, it is possible to intercept the response from the Identity Server and analyze the SAML message that has been sent to the Relying Party:

![intercepting requests with burp](/notsosaml/image-34.png)
And the decoded message:

![decoded SAML message](/notsosaml/image-35.png)
At this point, all we have to do is to inject an additional `<Assertion>` element that will contain just the admin username as a `<saml:NameID>` inside a `<saml:Subject>` element. The rest of the element will be the same as the original one.

![original one](/notsosaml/image-36.png)
![original object](/notsosaml/image-37.png)
![final object](/notsosaml/image-38.png)

**Note**: this flaw could be also exploited by using the "SAML Raider" extension on burp suite, configured with the "XSW 3" payload.
By sending the forged object, the following response should follow:

![response after injecting the payload](/notsosaml/image-39.png)
And, finally, we'll get admin access.

![admin access](/notsosaml/image-40.png)

## Spotting the bug
Spotting the vulnerability in the code is pretty trivial in this case. The request is hitting the miniorange_samlController.php and, afterwards, the MiniOrangeAcs.php#processSamlResponse method.

![bugged file #1](/notsosaml/image-41.png)
![bugged file #2](/notsosaml/image-42.png)

Since the signature check could be bypassed (the green mark on the screenshot), the signature of our injected <Assertion> object isn't checked and the current() function will get only the first element of the array, which is the injected object, ignoring the other ones.

Furthermore, the application doesn't enforce the <Assertion> signature and the x509 certificate.

## Conclusions
There are two main consequences for this bug:

- **Message Replay Attacks**: As seen from the screenshots, the plugin allows to send the SAML response directly to the relying party endpoint without using the Identity Provider. This allows an attacker to replay the authentication unlimited times.
- **Privilege Escalation Attacks**: An attacker could register a user using the external Identity Provider and escalate to higher privileges by knowing just the username of the target - which is pretty trivial since it is "admin", most of the time.

The affected version, at the time of writing, is the **8.x-2.22**, released on February 2021. The plugin could be downloaded from the [Drupal plugin site](https://www.drupal.org/project/miniorange_saml?ref=voidzone.me).

This exploit has been tested using Drupal version 9.1.10 with PHP 8.0.7 and Apache/2.4.38 (Debian) on a [Docker container](https://hub.docker.com/_/drupal/?ref=voidzone.me).

The main cause of this bug is the lack of enforcement for the x509 Certificate Value and Either SAML response or SAML assertion must be signed options in the Drupal admin interface. Unchecking these options could expose the entire Drupal application to the above mentioned vulnerabilities.

## Conclusions - afterwards
### Wordpress
The exploit isn't working for the Wordpress plugin version since it enforces the application to verify the signature of the <Assertion> and to provide a valid x509 certificate.

### Moodle
The faulty logic could be found in the moodle plugin codebase too. While i had no luck configuring the plugin in a correct way, the code looks vulnerable:

![moodle - bugged file](/notsosaml/image-44.png)

## Responsible disclosure timeline
- 06/16/2021: First bug notification to miniorange (xecurify) company
- 06/17/2021: Got first feedback, report has been sent
- 06/22/2021: Got first update, report will be reviewed in the next friday
- 07/01/2021: No updates. Sending the disclosure deadline
- 07/09/2021: Got update from Xecurify, expected behavior and "known" issue. A hotfix will be pushed in the next sprint
- 07/09/2021: Final disclosure
- 09/23/2021: SA published