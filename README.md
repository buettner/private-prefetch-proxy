# Private Prefetch Proxy Explained

Work-in-progress draft ([feedback & questions welcomed](https://github.com/buettner/private-prefetch-proxy/issues))

## Problem and motivation
There is [renewed interest](https://github.com/jeremyroman/alternate-loading-modes) in prefetching/prerendering as a way to improve loading performance on the web. However, prefetching cross-origin links reveals potentially identifiable information with the destination, e.g., the user’s cookies and IP address, before the user has explicitly signaled their interest in the site or content. 

Most of these concerns can be addressed with changes at the browser, e.g,. not sending cookies on prefetch requests, but hiding the client IP from the destination until the user navigates requires changes at the network level. While [technologies exist](https://developers.google.com/web/updates/2018/11/signed-exchanges) that enable prefetching without revealing the client IP to destinations, they require some [work to set up](https://developers.google.com/web/updates/2018/11/signed-exchanges#trying_out_signed_exchanges) which isn’t yet always [trivial](https://blog.amp.dev/2019/06/17/introducing-cloudflare-amp-real-url/).

To make it easier to achieve instant experiences on the web, we’re exploring an alternative design for privacy-preserving prefetch as described [in this blog post](https://blog.chromium.org/2020/12/continuing-our-journey-to-bring-instant.html). Key to the proposal is the use of an HTTP/2 CONNECT proxy to obfuscate the IP address from the destination site during prefetching, along with rules governing its usage and additional measures to ensure that the prefetches cannot be linked to the user.


## Goals
We want to make it easy for sites to take advantage of cross-origin prefetching without revealing the user’s IP address to the destination. We propose using an HTTP/2 CONNECT proxy to obfuscate the user’s IP during prefetching to achieve this goal, while at the same time maintaining the security properties of the web and giving both users and websites control over the use of the proxy.

## Non-goals
This proposal is only relevant for the cross-origin cases. For same-origin prefetching/prerendering, there is no need to hide the user’s IP address or other state. Indeed, the party triggering prefetch/prerender is the same party that is being prefetched/prerendered, and naturally has access to said information. While same-origin prefetch/prerendering are out-of-scope of this explainer, we are nevertheless interested in improving prefetching/prerendering for both cross-origin and same-origin scenarios through other efforts.

# FAQ
## TLS key leaks and private prefetch proxies

**Concern:** “TLS key leaks (e.g. heartbleed) pose a greater user threat with a private prefetch proxy because it allows an attacker to direct prefetches through a colluding proxy, thereby manipulating the network path and making MITM attacks easier.”

This is a risk if we allow websites to specify any “private prefetch proxy” of their choosing. For instance, a malicious website could specify their own proxy, loaded with compromised TLS keys, and trick the user into clicking a prefetched link for a legitimate website.

This suggests that “private prefetch proxies” need to be trusted by the browser before they will be used for prefetching. We are currently developing a “trusted private prefetch proxy” model with requirements and potentially audits.

## Risk of collusion
**Concern:** “Private prefetch proxies enable a new vector for cross-site user tracking, as the proxy can directly terminate TLS connections to origins the proxy owns or colludes with, and then it can directly add tracking identifiers to requests. ”

We believe that a “trusted private prefetch proxy” model would also address this concern. Not only must private prefetch proxies not introduce new  identifiers, they must in fact IP-blind all destination origins. 

## Learning about the user’s interests
**Concern:** “Prefetch proxies will learn about users based on their prefetch requests, which they could monetize  or leverage themselves.”

Similar to other concerns about the trustworthiness of the proxy, we believe that a trusted private prefetch proxy model will be sufficient to prevent such abuse.

In addition, both the referrer website and the user would have control over which proxies (if any) they are willing to use,  with prefetching being disabled if there is no agreement.

## Content blockers and extensions

**Concern:** “What about content blockers? How does this impact extensions?”

Browsers should continue to ensure that network requests and responses are subject to a user’s installed extensions even when the requests are handled by a private prefetch proxy.

For DNS based content blockers, there is a range of options to explore including allowing users to disable the feature altogether, or to enable an additional blocking DNS lookup for every domain at navigation time (along with the concomitant performance penalty).

## Impact for services provided by ISPs
**Concern:** “How does this interact with content filtering?”

We acknowledge that network administrators may need to filter content. We’re considering the following approach to avoid interfering in these scenarios. 

At startup and on a change of network, the browser would attempt to resolve a purpose-specific domain name, and examine the result:

 - If a response code other than NOERROR is returned (e.g. NXDOMAIN or SERVFAIL), or if a NOERROR response code is returned, but contains neither A nor AAAA records, then the browser would change its behavior in the following manner:

  - Upon navigation to a prefetched link, the browser would issue a blocking DNS lookup for the domain. This DNS lookup will happen at the same time and in the same manner as if the prefetch had not happened, providing the administrator with the same opportunity to filter content. 

## Abuse
**Concern:** “How will this interact with websites’ anti-abuse mechanisms?”

To protect against attackers using the proxy to abuse websites, the prefetch proxy must block traffic that does not fit the pattern of legitimate link prefetching e.g., based on the number of requests, session duration, etc.

We’re also considering potential schemes for authentication, as well as mechanisms by which website operators can opt-out of proxied prefetching (e.g., rejecting requests with the “Purpose: prefetch” header and potentially a blanket opt-out expressed in the DNS record). In addition, private prefetch proxies should allow for reverse DNS lookups of their IP addresses and publish an escalation path for help addressing potential abuse concerns. 

## Geolocation
**Concern:** "How will this work with geography-based use cases (e.g. Geo-filtering / Geo-access)?"

The destination server will see the IP of the proxy egress IP, not the user's IP; this will interfere with IP-based geolocation. 
Servers that rely on geolocation to determine what content to serve have the following options:
* Determine the location of the user at navigation time, e.g., by triggering a request via JS.
* Reject requests with the "Purpose: prefetch" header for resources that are georestricted.

More speculative ideas worth exploring are:
* Requiring proxies to only egress traffic from IPs in the same country/region as the user. The challenge here is having agreement on the granularity of "region", as proxies likely can't egress in every country.
* APIs/mechanism by which the proxy can tell the destination what general region the user is in. Similar to the above, there would need to be agreement about the required granularity.

## Trusted Private Prefetch Proxies (TPPP)
**Question**: “What are ‘trusted private prefetch proxies’?”

We would like to firm this up with the help of the community. 

At a high level, here is a tentative and non-exhaustive list of aspects that we think would be needed:

 - Requirements to define expected behavior: “a TPPP must hide the IP address of the prefetch requester”, what data can be logged, retention policy for those logs, etc.
 - Usage rules (e.g. abuse prevention).
 - Potentially audits to assert that a TPPP is implementing the requirements and usage rules as specified.


## Other concerns or questions?

Please [file an issue](https://github.com/buettner/private-prefetch-proxy/issues) if you have identified something that ought to be addressed in this section.


# Feedback, discussion


If you are interested in this proposal, please consider participating in existing [discussions](https://github.com/buettner/private-prefetch-proxy/issues) or filing new issues to share feedback or ask questions. In addition, if you are interested in prefetching/prerendering in general, you might be interested in [discussions for alternate loading modes](https://github.com/jeremyroman/alternate-loading-modes/issues) as well.

