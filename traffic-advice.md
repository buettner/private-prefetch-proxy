# Traffic Advice

Publishers may wish not to accept traffic from [private prefetch proxies](README.md) and other sources other than direct user traffic, for instance to reduce server load due to speculative prefetch activity.

We propose a well-known "traffic advice" resource, analogous to `/robots.txt` (for web crawlers), which allows an HTTP server to declare that implementing agents should stop sending traffic to it for some time.

## Proposal

Agents which respect traffic advice should fetch the well-known path `/.well-known/traffic-advice.json`. If it returns a `200 OK` response with the `application/json` MIME type, the response body should contain valid JSON like the following:

```json
[
    {"user_agent": "prefetch-proxy", "disallow": true}
]
```

Each agent has a series of identifiers it recognizes, in order of specificity:
* its own agent name
* decreasingly specific generic categories that describe it, like `"prefetch-proxy"`
* `"*"` (which applies to every implementing agent)

It finds the most specific element of the response, and applies the corresponding advice (currently only a boolean which advises disallowing all traffic) to its behavior. The agent should respect the cache-related response headers to minimize the frequency of such requests and to revalidate the resource when it is stale.

Currently the only advice is the key `"disallow"`, which specifies a boolean which, if present and `true`, advises the agent not to establish connections to the origin. In the future other advice may be added.

If the response has a `404 Not Found` status, on the other hand, the agent should apply its default behavior.

## Why not robots.txt?

`robots.txt` is designed for crawlers, especially search engine crawlers, and so site owners have likely already established robots rules because they wish to limit traffic from crawlers -- even though they have no such concern about prefetch proxy traffic. The `robots.txt` format is also designed to limit traffic by path, which isn't appropriate for agents which do not know the path of the requests they are responsible for throttling (as with a CONNECT proxy carrying TLS traffic).

A more similar textual format would be possible, but the format for parsing `robots.txt` is not consistently specified and implemented. By contrast, JSON implementations are widely available on a wide variety of platforms used by site owners and authors.