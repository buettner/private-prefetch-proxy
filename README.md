# Private Prefetch Proxy Explained

Work-in-progress draft ([feedback & questions welcomed](https://github.com/buettner/private-prefetch-proxy/issues))

## Problem and motivation
There is [renewed interest](https://github.com/jeremyroman/alternate-loading-modes) in prefetching/prerendering as a way to improve loading performance on the web. However, prefetching cross-origin links reveals potentially identifiable information with the destination, e.g., the user’s cookies and IP address, before the user has explicitly signaled their interest in the site or content. 

Most of these concerns can be addressed with changes at the browser, e.g,. not sending cookies on prefetch requests, but hiding the client IP from the destination until the user navigates requires changes at the network level. While [technologies exist](https://developers.google.com/web/updates/2018/11/signed-exchanges) that enable prefetching without revealing the client IP to destinations, they require some [work to set up](https://developers.google.com/web/updates/2018/11/signed-exchanges#trying_out_signed_exchanges) which isn’t yet always [trivial](https://blog.amp.dev/2019/06/17/introducing-cloudflare-amp-real-url/).

To make it easier to achieve instant experiences on the web, we’re exploring an alternative design for privacy-preserving prefetch as described [in this blog post](https://blog.chromium.org/2020/12/continuing-our-journey-to-bring-instant.html). Key to the proposal is the use of an HTTP/2 CONNECT proxy (or potentially in the future, a [QUIC proxy](https://tools.ietf.org/html/draft-pauly-masque-quic-proxy-00)) to obfuscate the IP address from the destination site during prefetching, along with rules governing its usage and additional measures to ensure that the prefetches cannot be linked to the user.

## Goals
We want to make it easy for sites to take advantage of cross-origin prefetching without revealing the user’s IP address to the destination. We propose using an HTTP/2 CONNECT proxy to obfuscate the user’s IP during prefetching to achieve this goal, while at the same time maintaining the security properties of the web and giving both users and websites control over the use of the proxy.

## Non-goals
This proposal is only relevant for the cross-origin cases. For same-origin prefetching/prerendering, there is no need to hide the user’s IP address or other state. Indeed, the party triggering prefetch/prerender is the same party that is being prefetched/prerendered, and naturally has access to said information. While same-origin prefetch/prerendering are out-of-scope of this explainer, we are nevertheless interested in improving prefetching/prerendering for both cross-origin and same-origin scenarios through other efforts.

## Challenges
There are two primary concerns we see with prefetch proxies. The first is that they can amplify the impact of compromised TLS certificates. Today, an attacker must have a compromised TLS certificate and also be on the network path between the user and that origin to MITM the connection. If attackers can designate and run prefetch proxies, they can trivially put themselves on the network path. A related concern is collusion, where a prefetch proxy works with some destinations to selectively unblind requests.

Today, there are no technical means by which a browser can verify that a CONNECT proxy is not MITMing the client->origin TLS connection. While confidential computing technology may allow the browser to verify exactly what code is being run by the proxy, the technology is in its infancy. 

The second concern is that the proxy may become an aggregation point for user data. The set of links shown to a user on a referring site is both business data of the referrer and potentially PII for the user (e.g., if they are logged into the referring site). Of course, browsers see links on referrer pages today, but most browsers do not make that data available to backend infrastructure. Prefetch proxies should not build profiles of users or referring sites.

This leads to the core challenge of making this feature available to as much of the web as possible while:
1. Giving users and referrers control over who they trust with their data.
1. Giving publishers the ability to opt-out of the feature.
1. Giving browsers the ability to ensure that prefetch proxies do not put their users at risk of TLS attacks or tracking.

## Exploration
In the interest of starting a discussion with the community, we are sharing a tentative plan for making privacy-preserving prefetching more accessible and appealing to a broad set of parties. We would [love to hear](https://github.com/buettner/private-prefetch-proxy/issues) your questions, concerns and feedback to refine our thinking!

### Browser
A key responsibility of a browser is to ensure the safety of its users. As noted in the [Challenges](https://github.com/buettner/private-prefetch-proxy#challenges) section, the primary concern with private prefetch proxies is that they introduce safety risks which the browser can not sufficiently address with currently available technology. Until this gap is resolved, it follows that the browser should only use  a private prefetch proxy that it implicitly trusts, either because it is  operated by the browser vendor itself, or by a third party under a contract (i.e. similar to the approach taken with DoH and VPN features by some browser vendors).

### Referrers and users
Outgoing links on a referring page may be the result of information the referrer knows about the user.  Consequently, outgoing links should not be prefetched, even with privacy preserving guarantees, without at least one of those two parties’ consent  (and the other party should be able to opt-out). 

### Publishers
Some publishers may not be comfortable with proxied prefetches even if the referrer, user and browser all are. A publisher’s level of comfort might depend on: the browser (and which proxy by association), user related characteristics (e.g. geo restrictions), or perhaps the ability of a given referrer to balance performance and data usage. We would love input on other potential concerns to help us prioritize refinements. Please [**share concrete details as to why**](https://github.com/buettner/private-prefetch-proxy/issues) a particular aspect matters to you.

### Participation model
We believe that a referrer opt-in combined with opt-outs for users and publishers provides the best value and flexibility to all parties involved:
 * Referrers can choose to opt-in if they believe the benefits are worth it.
 * Both users and publishers can choose to opt-out if they are uncomfortable with the feature (e.g. concerned about extra data usage).

#### Referrer opt-in
Referrers opt-in to the feature by indicating which links should be prefetched. Tentatively, we suggest the following approach built on top of a minimal subset of the speculation rules from the [Alternate Loading Modes](https://github.com/jeremyroman/alternate-loading-modes/blob/main/triggers.md) proposal:

```html
<script type="speculationrules">
{
  "prefetch": [
    {"source": "list",
     "urls": ["https://whizbang.example/bestof2020.html"],
     "requires": ["anonymous-client-ip"]}
  ]
}
</script>
```

Where:
  - `urls` would contain a list of URLs the referrer believes to be good candidates for prefetching. The browser would consider this list for prefetching, in addition to  other constraints (e.g. bandwidth, prioritizing the main user experience, user preferences, etc).
 - `requires": ["anonymous-client-ip"]` indicates that the referrer wants the cross origin prefetches to be done in a privacy preserving manner.


#### User opt-out
Users can opt-out of the feature at any time. Furthermore, users can temporarily opt-out of the feature by using their browser’s private browsing mode.

#### Publisher opt-out
One option for origin-wide opt-out is to leverage the publisher's DNS record:
* Publishers specify in their DNS entry that they are opting out of proxied prefetching (completely or with some TBD granularity if necessary). 
* The DNS check would be done by the proxy for privacy reasons;  issuing a DNS request from the browser before navigation would share prefetch information with the DNS resolver and potentially the target host. 

Alternatively (or in addition), we could define a [/.well-known URL](https://tools.ietf.org/html/rfc5785) that can be used for publisher opt-out, and this URL would be fetched and cached by the proxy. This has the advantage that it is easier for developers to add a new resource than to modify their DNS record. 

Ideally, the browser would fetch the opt-out signal *before* making a connection to the proxy. While there are proposals to enable anonymous fetching of both DNS records ([Oblivious DNS](https://tools.ietf.org/html/draft-pauly-dprive-oblivious-doh-00)) and HTTP resources ([Oblivious HTTP](https://tools.ietf.org/html/draft-thomson-http-oblivious-00)), neither is well-supported yet. If either of those proposals gains traction, we may want to revisit the publisher opt-out design to take advantage of Oblivous fetching.

In addition, publishers can opt-out for individual requests, for example,  when dealing with temporary traffic spikes or other issues. For these, publishers should look for the `Purpose: prefetch` request header and reject requests accordingly (see [Geolocation](https://github.com/buettner/private-prefetch-proxy#geolocation) for an example use case).

### Future opportunities
We’re continuing to explore ways to safely prefetch via proxies not operated by the browser. In that case, referrers may wish to specify which (if any) proxies they trust with their user data. The *speculation rules* approach offers a flexible pattern which would allow for this extension.

If you have ideas on future opportunities or want to suggest a different approach, please [start a topic](https://github.com/buettner/private-prefetch-proxy/issues). Thanks!


## Prefetching Details
### Using an isolated network context
Prefetches should not reveal any local state that can be used to identify the user. The CONNECT proxy masks the IP address, but the browser is responsible for not revealing other information that can be used to identify the user. 

Specifically:
* Cookies must not be sent on prefetches.
* Prefetches must use an isolated network context that does not reveal state from the HTTP cache, previous TLS sessions, etc.
* Static fingerprinting surfaces such as User-Agent must be bucketed, e.g., only specifiying the major version of the browser.

In addition, prefetches should not persist any state (cookies, HTTP caching) unless the user navigates to the prefetched link. 

### What to prefetch
Our experiment found that fetching the mainframe HTML, along with statically linked CSS and synchronous Javascript, provided a 40% LCP improvement at the median. Fetching other resources, for example images, may further improve user experience at the cost of more wasted bytes on mispredictions. All prefetches carry the "Purpose: prefetch" header so origins can identify them. 

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

## Traffic analysis
**Question**: "Even though prefetches are end-to-end encrypted between the browser and the destination, can't the proxy perform traffic analysis attacks?"

[By design](https://github.com/buettner/private-prefetch-proxy#using-an-isolated-network-context), prefetches should not reveal any local state to the destination that could be used to identify the user. This means that the responses cannot be personalized. The proxy could learn, for example, that the destination runs A/B experiments on non-logged in users. But we don't believe this information is particularly valuable, and the destination can always reject prefetch requests. 

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

