# HSTS Tracking Prevention Explainer

## Participate
- https://github.com/explainers-by-googlers/HSTS-Tracking-Prevention/issues

## Background

The HTTP Strict Transport Security (HSTS) response header field, specified by [RFC 6797](https://datatracker.ietf.org/doc/html/rfc6797), is a mechanism that sites can use to tell the browser not to load a host name over insecure HTTP but to instead use HTTPS. Once a hostname is saved the browser will always request any resources at that host over HTTPS.

### Terminology
For this document the terms “HSTS registration”, or similar, means that an HTTP response includes a valid HSTS header and the browser saves the information to its HSTS cache.

## Problem

The browser now saves the presence of a host name on its internal HSTS cache. Since a host name is either present or not this is equivalent to a single bit of information, a boolean. 

Suppose that a third-party wanted to store 32-bits of information, enough to uniquely identify a user. That third-party could embed a script in, for example, a news site. That script would attempt to load 32 images from the third-party site all over HTTP.
E.x.: http://bit0.example.com/image.jpg, http://bit1.example.com/image.jpg, …, http://bit31.example.com/image.jpg

The server could then redirect a portion of those requests to HTTPS and return an HSTS header. Now the next time the user visits the news site and the third-party’s script tries to load the images, the user’s browser will make some requests to the HTTP version and the rest to the HTTPS version, essentially sending a 32-bit identifier back to the third-party. If the third-party’s script is used on multiple different sites then this gives the third-party the ability to track the user around the web.

## Proposal

We will only apply HSTS upgrades to top-level navigations.

By not applying HSTS upgrade to any sub-resources it will not be possible for the stored identity to be read unless the browser is navigated to every url, adding significant time and friction to any identification attempts. We will continue to accept both first and third-party HSTS registrations.

This proposal is an implementation of [What if HSTS only applied to top-level navigations?](https://github.com/mikewest/strict-navigation-security) 

## Enablers

While this change could, by itself, reduce security for users we fortunately have a number of other behaviors working in our favor that greatly reduce any potential danger.

### Mixed Content Upgrading/Blocking

Mixed content is insecure HTTP sub-resources being loaded on a secure HTTPS page.

Since Chrome 81 we have been completely upgrading or blocking mixed content. Meaning that on secure pages HSTS is effectively obsolete for sub-resources.

### Automatic HTTPS Navigation Upgrades

In instances where a site hasn’t been added to the browser’s HSTS cache Chrome will still attempt to automatically upgrade the connection to HTTPS. See [Chromium Blog: Towards HTTPS by default](https://blog.chromium.org/2023/08/towards-https-by-default.html).

### HTTPS-First Mode

Once enabled, [HTTPS-First Mode](https://blog.chromium.org/2023/08/towards-https-by-default.html) will prevent an active network attacker from silently downgrading a connection's security. This strengthens the security improvements offered by navigation upgrades which could otherwise be downgraded.

### HTTPS Usage

HTTPS is now [widely used](https://transparencyreport.google.com/https/overview?hl=en) and the vast majority of pages are loaded over HTTPS. Coupled with HTTPS-First Mode and mixed content upgrading/blocking this means that the vast majority of pages do not need HSTS. 

### SameSite cookies are scheme aware

Cookies with `SameSite=none` have required `Secure` since [Chrome 80](https://developers.google.com/search/blog/2020/01/get-ready-for-new-samesitenone-secure#chrome-enforcement-starting-in-february-2020) and the notion of “same-site” has considered the scheme since [Chrome 89](https://chromestatus.com/feature/5096179480133632) (for cookies).

Both of these features help to limit the leakage of state between HTTP and HTTPS versions of a site.

Looking forward, if [Origin-Bound Cookies](https://chromestatus.com/feature/4945698250293248) launches then the HTTP and HTTPS version of a site will have their cookies completely separated, preventing any kind of state leakage.

## HTTP Top Level

With our proposed solution, sub-resources within a page loaded over HTTP would have a degraded experience; these sub-resources would never be upgraded.

This could leave previously upgradable resources open to a MITM attack, potentially degrading the user’s security. Thankfully less than ~5% of pages are loaded via HTTP. Moreover an attacker can always arbitrarily alter a resource request over the top-level page’s HTTP connection. Overall the security impact is expected to be limited.


## Alternatives Considered 
### Blocking Third-Party Registrations

This alternative would have prevented third-party resources from registering with the HSTS in order to limit tracking. Sub-resource requests would be eligible for an HSTS upgrade.

This option was not selected because Chrome mixed content upgrading/blocking means that HSTS upgrades have no meaningful effect on sub-resources and so this alternative provides no benefit.

### Partitioning

This alternative would partition the HSTS cache entries by the top-level site. Doing so would limit any identity information to only the top-level site that it was originally set on, thus preventing tracking between sites.

This option was not selected because our proposed solution has comparable security benefits and allows HSTS registrations from third-parties to apply to any top-level navigations.

## Prior Art

Other browsers, Safari and Firefox, have previously implemented their own version of HSTS tracking preventions. While these designs differ from ours, they nonetheless informed our approach.
 
* WebKit - [Protecting Against HSTS Abuse | WebKit](https://webkit.org/blog/8146/protecting-against-hsts-abuse/)
* Mozilla - [Bug 1701192 - don't allow third-party loads to set HSTS state](https://hg.mozilla.org/integration/autoland/rev/9bae0f6ea847) 

