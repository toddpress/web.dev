---
title: HTML
authors:
  - imkevdev
description: Every website starts with a request for an HTML document, and how you structure your HTML's dependent resources such as CSS and JavaScript has a big role to play in how fast your website loads. In this module, you'll learn important concepts such as HTML caching, parser blocking, render blocking, and more.
date: 2023-09-01
tags:
  - performance
---

The first step in building a website that loads quickly is to receive a timely response from the server. When you enter a website's URL in the address bar, the browser sends a [`GET` request](https://developer.mozilla.org/docs/Web/HTTP/Methods/GET) to the server for that resource. The first request a browser sends to retrieve a web page is most often for an HTML resource—and ensuring that HTML arrives quickly with minimal delay is a key performance goal.

The request goes through several steps. Reducing the number of steps or the time spent on each step will give you a faster Time to First Byte ([TTFB](/ttfb/)). TTFB is not a user-centric metric, but a slow TTFB makes it challenging to reach the designated "good" thresholds for metrics such Largest Contentful Paint ([LCP](/lcp/)) and First Contentful Paint ([FCP](/fcp/)).

{% Aside %}
For more information on how to optimize your website's TTFB, read the [optimize TTFB guide](/optimize-ttfb/).
{% endAside %}

## Minimize Redirects

When a resource is requested, the server may respond with a redirect, either with a permanent redirect (301 response) or a temporary one (302 response). Redirects will slow down page load speed because it requires the browser to make another HTTP request at the new location to retrieve the resource.

There are two types of redirects:

1. _Same-origin redirects_ that occur within your website. These types of redirects are completely within your control.
2. _Cross-origin redirects_ that are initiated by another origin. These types of redirects are typically out of your control.

Cross-origin redirects are used by ads, URL-shortening services, and other third parties. Though cross-origin redirects are outside of your control, you may still want to check that you avoid multiple redirects—for example, having an ad that links to an HTTP page which in turn redirects to its HTTPS equivalent, or a cross-origin redirect that arrives to your origin, but then triggers a same-origin redirect.

{% Aside %}
A common same-origin redirect pattern is to redirect users from a URL ending in a trailing slash to a non-trailing slash equivalent or vice-versa—for example, redirecting the user from `example.com/page/` to `example.com/page`. When creating internal links between your pages, you want to avoid linking to a page that will respond with a redirect and link directly to the correct location.
{% endAside %}

## Caching HTML responses

Caching HTML responses is difficult, since the response includes links to other critical resources such as CSS, JavaScript, and images. These resources may include [a unique fingerprint](https://bundlers.tooling.report/hashing/) which changes based on a file's contents. This means that your cached HTML document may become stale following a deployment, as it would contain links to outdated resources. Nonetheless, a short cache lifetime—rather than no caching—can have benefits such as allowing a resource to be cached at a CDN—reducing the number of requests that are served from the origin server—and in the browser, allowing resources to be revalidated rather than downloaded again.

If your HTML page's content is personalized somehow—such as for an authenticated user—you probably do not want to cache content for security reasons. If the HTML response is cached on the user's browser, you are unable to invalidate the cache. Therefore, it's best to avoid caching HTML in these cases.

A conservative approach to HTML caching could be to use the [`ETag`](https://developer.mozilla.org/docs/Web/HTTP/Headers/ETag) or [`Last-Modified`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Last-Modified) response headers. An ETag (or entity tag) header is an identifier that uniquely represents the requested resource.

```http
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

Whenever the resource changes, a new `ETag` value must be generated. On subsequent requests, the browser sends the `ETag` value through the [`If-None-Match` request header](https://developer.mozilla.org/docs/Web/HTTP/Headers/If-None-Match). If the `ETag` on the server matches the one sent by the browser, the server will respond with a `304 Not Modified` response code, and the browser will use the resource from the cache.

The downside with ETags or the `Last-Modified` header is that the browser still needs to make a network request to revalidate the freshness of the cached resource. So while the 304 response is smaller than the full payload and reduces download time, you still incur the costs to open a connection to the server and wait for a response, which still incurs some amount of latency.

{% Aside %}
The `Last-Modified` header works similarly by including a response header with the date and time when the resource was last updated.
{% endAside %}

{% Glitch 'learn-performance-caching' %}

## Server Response Times

If the response is not cached, the server response time is highly dependent on your hosting provider and backend stack. A web page that serves a dynamically generated response—such as fetching data from a database, for example—may well have a higher TTFB than a static web page that can be served immediately without significant compute time on the backend. Serving a loading spinner and then fetching all data on the client side moves the effort from a more predictable server-side environment to a potentially unpredictable client-side one. Minimizing client-side effort usually results in improved user-centric metrics.

If you are experiencing a slow TTFB, you can expose information on where time was spent on the server through the use of the [`server-timing` response header](https://developer.mozilla.org/docs/Web/HTTP/Headers/Server-Timing).

```http
Server-Timing: fetch;dur=55.5, encoding;dur=220
```

A `Server-Timing` header can include multiple metrics and the duration for each one. In the example above, the response header includes two timings for fetching and encoding, with durations of 55.5 milliseconds and 220 milliseconds respectively.

{% Aside %}
You can find out more information on the `Server-Timing` response header in the [Optimize TTFB guide](/optimize-ttfb/#understanding-high-ttfb-with-server-timing).
{% endAside %}

You may also want to review your hosting infrastructure and confirm that you have adequate resources to handle the traffic your website is receiving. Shared hosting providers are often susceptible to a high TTFB and alternative solutions may be more costly.

{% Aside 'important' %}
You can compare the TTFB of popular hosting providers at [ismyhostfastyet.com](https://ismyhostfastyet.com/). The data is made up of real user experiences collected from the [Chrome User Experience Report (CrUX)](https://developer.chrome.com/docs/crux/) dataset.
{% endAside %}

## Compression

Text-based responses such as HTML, JavaScript, CSS, and SVG should be compressed to reduce their transfer time over the network. The most widely used compression algorithms are gzip and Brotli. Brotli results in about a 15% to 20% improvement over gzip.

## Content Delivery Networks (CDNs)

A [Content Delivery Network (CDN)](/content-delivery-networks/) is a distributed network of servers that cache resources from your origin server, and in turn serves them from edge servers that are physically closer to your users. The physical proximity to your users reduces [round-trip time (RTT)](https://en.wikipedia.org/wiki/Round-trip_delay), while optimizations such as HTTP/2 or HTTP/3, caching, and compression allow the CDN to serve content more quickly than if it would be fetched from your origin server. Utilizing a CDN can significantly improve your TTFB.

{% Aside %}
For an in-depth look at CDNs and their benefits, read the [CDN guide](/content-delivery-networks/).
{% endAside %}
