# Traffic Advice

Publishers may wish not to accept traffic from [private prefetch proxies](README.md) and other sources other than direct user traffic, for instance to reduce server load due to speculative prefetch activity.

We propose a well-known "traffic advice" resource, analogous to `/robots.txt` (for web crawlers), which allows an HTTP server to declare that implementing agents should stop sending traffic to it for some time. The formal traffic-advice specification can be found [here](https://buettner.github.io/private-prefetch-proxy/traffic-advice.html).

## Proposal

HTTP request activity can broadly be divided into:
* activity on behalf of a user interaction (e.g., a web browser a web page requested by the user), or which for another reason cannot easily be discarded
* activity for which there is an existing specialized mechanism for throttling traffic (e.g. web crawlers respecting `robots.txt`)
* activity which can easily be discarded (e.g., because it corresponds to a prefetch which improves loading performance but not correctness) at the server's request (e.g., because it is under load or the operator otherwise does not wish to serve non-essential traffic)

Applications in the third category should consider acting as *agents which respect traffic advice*, so as to respect the server operator's wishes with a minimum resource impact.

Agents which respect traffic advice should fetch the well-known path `/.well-known/traffic-advice`. If it returns a response with an [ok status](https://fetch.spec.whatwg.org/#ok-status) and a `application/trafficadvice+json` MIME type, the response body should contain valid UTF-8 encoded JSON like the following:

```json
[
    {"user_agent": "prefetch-proxy", "disallow": true}
]
```

Each agent has a series of identifiers it recognizes, in order of specificity:
* its own agent name (e.g. `"ExamplePrivatePrefetchProxy"`)
* decreasingly specific generic categories that describe it, like `"prefetch-proxy"`
* `"*"` (which applies to every implementing agent)

It finds the most specific element of the response, and applies the corresponding advice (currently only a boolean which advises disallowing all traffic) to its behavior. The agent should respect the cache-related response headers to minimize the frequency of such requests and to revalidate the resource when it is stale.


If the response has a `404 Not Found` status (or a similar status), on the other hand, the agent should apply its default behavior.

## Why not robots.txt?

`robots.txt` is designed for crawlers, especially search engine crawlers, and so site owners have likely already established robots rules because they wish to limit traffic from crawlers -- even though they have no such concern about prefetch proxy traffic. The `robots.txt` format is also designed to limit traffic by path, which isn't appropriate for agents which do not know the path of the requests they are responsible for throttling (as with a CONNECT proxy carrying TLS traffic).

A more similar textual format would be possible, but the format for parsing `robots.txt` is not consistently specified and implemented. By contrast, JSON implementations are widely available on a wide variety of platforms used by site owners and authors.

## Application to private prefetch proxies

For example, suppose a private prefetch proxy, `ExamplePrivatePrefetchProxy`, would like to respect traffic advice in order to allow site owners to limit inbound traffic from the proxy.

When a client of the proxy service (e.g., a web browser) requests a connection to `https://www.example.com`, the proxy server issues an HTTP request for `https://www.example.com/.well-known/traffic-advice`. It receives the sample response body from above. It recognizes `"prefetch-proxy"` as the most specific advice to apply to itself.

It caches this result (traffic is presently disallowed) at the proxy server (or even across multiple proxy server instances run by the same operator), and refuses client connections to `https://www.example.com` until an updated `/.well-known/traffic-advice` resource no longer disallows traffic. Even if a large number of proxy clients request connections to `https://www.example.com`, the site operator and its CDN do not receive traffic from the proxy except for infrequent requests to revalidate the traffic advice (which may be, for example, once per hour).
