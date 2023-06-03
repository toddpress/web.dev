---
title: Assisting the browser with resource hints
authors:
  - imkevdev
  - jlwagner
description: Resource hints are a collection of features available in HTML that can assist the browser in loading resources earlier, or with higher priority. In the module, you'll learn about a few resource hints that can help your page's load even faster.
date: 2023-09-01
tags:
  - performance
---

Developers can optimize the page's load time by informing the browser how to load and prioritize resources. Resource hints were the first of these, but preload, Fetch Priority API and lazy loading have followed in recent years to further enhance those capabilities.

Resource hints instruct the browser to perform certain actions ahead of time that could improve loading performance. Resource hints can perform actions such as performing early DNS lookups, connecting to servers ahead of time, and even fetching resources before the browser would ordinarily discover them.

Resource hints may be specified in HTML, or set as an HTTP header. For the scope of this module, we'll be looking at [`preconnect`](https://www.w3.org/TR/resource-hints/#dfn-preconnect) and [`preload`](https://developer.mozilla.org/docs/Web/HTML/Link_types/preload). 

## `preconnect`

```html
<link rel="preconnect" href="https://example.com">
```

The `preconnect` hint is used to establish a connection to another origin from where you are fetching critical resources. For example, you may be hosting your images or assets on a CDN domain.

By using `preconnect`, you anticipate that the browser will be connecting to a specific cross-origin server, and open that connection before waiting for the HTML parser or preload scanner to do so. Using `preconnect` is generally inexpensive, but preconnecting to too many cross-origins may hinder performance, so use it sparingly.

A [common use-case for `preconnect` is Google Fonts](https://almanac.httparchive.org/en/2021/resource-hints#fig-14). Google Fonts recommends that you `preconnect` to the `https://fonts.googleapis.com` domain that serves the `@font-face` declarations and to the `https://fonts.gstatic.com` domain that serves the font files.

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

The `crossorigin` attribute is used to indicate whether a resource must be fetched using [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/docs/Web/HTTP/CORS). When using the `preconnect` hint, if the resource being downloaded from the origin uses CORS—such as font files—then you need to add the `crossorigin` attribute to the `preconnect` hint. If you omit the `crossorigin` attribute the browser will open a new connection when it downloads the font files and not reuse the connection opened using the `preconnect` hint.

## `preload`

The `preload` directive is used to initiate a request for a resource required for rendering the page:

```html
<link rel="preload" href="/lcp-image.jpg" as="image">
```

`preload` hints should be limited to late-discovered critical resources. The most common use cases are font files, CSS files fetched through `@import` declarations, or CSS `background-image` resources that are determined to be [Largest Contentful Paint (LCP) candidates](/lcp/#what-elements-are-considered). In such cases, these files would not be discovered by the preload scanner as the resource is referenced in the CSS file.

{% Aside 'caution' %}
If you are using `preload` to download a CSS `background-image` or an `img` that varies based on the viewport, you'll need to add the [`imagesrcset` attribute to the `preload` hint to download the correct image](/preload-responsive-images/). You should also exclude the `src` attribute so browsers that do not support responsive preloading do not download the fallback image.
{% endAside %}

Similarly to `preconnect`, the `preload` directive requires the `crossorigin` attribute if you are preloading a CORS resource—such as fonts. If you do not add the `crossorigin` attribute—or add it for non-CORS requests—then the resource will be downloaded twice, wasting bandwidth that could have been better spent on other resources.

```html
<link rel="preload" href="/font.woff2" as="font" crossorigin>
```

{% Aside 'important' %}
A resource specified in a `preload` directive will also be downloaded twice if the `as` attribute is missing on the directive's `<link>` element. For a list of `as` attribute values, consult [this MDN article](https://developer.mozilla.org/docs/Web/HTML/Attributes/rel/preload#what_types_of_content_can_be_preloaded).
{% endAside %}

In the example above, the browser is instructed to preload `/font.woff2` using a CORS request—even if `/font.woff2` is on the same domain.

{% Aside 'caution' %}
The `preload` directive is a very powerful performance optimization—one that can _overused_, in fact. Resources downloaded via the `preload` directive will effectively downloaded at high priority, and if overused, `preload` may create bandwidth contention and could even slow down page loads.
{% endAside %}

## `prefetch`

The `prefetch` directive is used to initiate a low-priority request for a resource that will be used for future navigations:

```html
<link rel="prefetch" href="/next-page.css" as="style">
```

This directive largely follows the same format as the `preload` directive, only the `<link>` element's `rel` attribute uses a value of `"prefetch"` instead. Unlike the `preload` directive, however, `prefetch` is largely speculative in that you're initiating a fetch for a resource for a future navigation that may or may not happen.

There are times when `prefetch` can be beneficial—for example, if you've identified a user flow on your website that most users follow to completion, a `prefetch` for a render-critical resource for those future page can help to reduce load times for that page, as the resource will already be in the browser cache when the user navigates to it.

{% Aside 'caution' %}
Given the speculative nature of `prefetch`, its use comes with the potential downside that data used to fetch the resource may go unused if the user does not navigate to the page that ends up needing the prefetched resource. Rely on your analytics or other data sources for your website's usage patterns to decide for yourself if a `prefetch` is a good idea. Alternatively, you can use [the `Save-Data` hint](/optimizing-content-efficiency-save-data/#detecting-the-save-data-setting) to opt out of prefetches for users who have specified a preference for reduced data usage.
{% endAside %}

## Fetch Priority API

On Chromium-based browsers, you can use the [`Fetch Priority API`](/fetch-priority/) using the `fetchpriority` attribute to increase the priority of a resource. You can use the attribute with `link`, `img`, and `script` tags.

```html
<div class="gallery">
  <div class="poster">
    <img src="img/poster-1.jpg" fetchpriority="high">
  </div>
  <div class="thumbnails">
    <img src="img/thumbnail-2.jpg" fetchpriority="low">
    <img src="img/thumbnail-3.jpg" fetchpriority="low">
    <img src="img/thumbnail-4.jpg" fetchpriority="low">
  </div>
</div>
```

{% Aside 'important' %}
The `fetchpriority` attribute is [particularly effective](/fetch-priority/#increase-the-priority-of-the-lcp-image) when used a page's LCP image. By raising the priority of an LCP image with this attribute, you can improve a page's LCP with little effort.
{% endAside %}

By default, images are fetched with a **Low** priority. After layout, if the image is found to be within the initial viewport, the priority is increased to **High** priority. In the example above, `fetchpriority` immediately tells the browser to download the larger LCP image with a **High** priority, while the less important thumbnail images are downloaded with a lower priority.

Modern browsers load resources in two phases. The first phase is reserved for critical resources and ends once all blocking scripts have been downloaded and executed. During this phase, **Low** priority resources are only downloaded if there are less than two in-flight requests at the time the resources are discovered. By using `fetchpriority="high"` you can increase the priority of a resource, enabling the browser to download it during the first phase.

## Browser hints demos

{% Glitch 'learn-performance-resource-hints' %}
