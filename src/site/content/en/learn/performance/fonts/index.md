---
title: Fonts
authors:
  - imkevdev
description: TODO
date: 2023-09-01
tags:
  - performance
---

In the previous modules, we discussed optimizing HTML, CSS, JavaScript, and media resources. In this module, we will go through some methods to optimize font resources.

Fonts can impact page performance at both load time and rendering time. Large font files can take a while to download and negatively affect [First Contentful Paint (FCP)](/fcp/), while the incorrect [`font-display` value](https://developer.mozilla.org/docs/Web/CSS/@font-face/font-display) could cause an undesirable layout shifts that contribute to a page's [Cumulative Layout Shift (CLS)](/cls).

## Discovery

A page's web fonts are defined in a stylesheet using a `@font-face` declaration:

```css
@font-face {  
  font-family: "Open Sans";
  src: url("/fonts/OpenSans-Regular-webfont.woff2") format("woff2");
}
```

The above snippet is defining a `font-family` named `"Open Sans"` and tells the browser where to find the respective font file. To conserve bandwidth the browser does not download the font file until it is determined that the current page requires it in its presentation.

```css
h1 {  
  font-family: "Open Sans";
}
```

In the example above, the browser will download the `"Open Sans"` font file when it encounters an `<h1>` element.

### `preload`

If your `@font-face` declarations are defined in an external stylesheet, the browser can only begin downloading them after it has downloaded the stylesheet. This makes font files a late-discovered resource.

You can initiate an early request for the font files by using the `preload` directive. The `preload` directive will make the font files discoverable early during page load, and the browser will begin downloading them without waiting for the stylesheet to finish downloading and parsing. The `preload` directive does not wait until the font is needed on the page.

```html
<link rel="preload" as="font" href="/fonts/OpenSans-Regular-webfont.woff2" crossorigin>
```

{% Aside %}
The `preload` directive should be used carefully. Overuse of the `preload` directive may take away bandwidth from other critical resources, and if used excessively, you may be downloading fonts that are not needed for the current page.
{% endAside %}

### Inline `@font-face` declarations

You can make fonts discoverable earlier during page load by inlining render-blocking CSS—including the `@font-face` declarations—in the `<head>` of your HTML. In this case, the browser will discover the fonts earlier in the page load as it does not need to wait for an external stylesheet to download.

{% Aside 'caution' %}
The browser will only begin downloading font files after all render-blocking CSS has been loaded. This means that if you have inlined your `@font-face` declarations, but the remaining CSS is in an external stylesheet, the browser will still have to wait for the external stylesheet to be downloaded.
{% endAside %}

Inlining `@font-face` declarations have an advantage over using the `preload` hint, as the browser will only download the fonts necessary to render the current page. This eliminates the risk of downloading unused fonts.

{% Aside 'important' %}
Inlining font files themselves into your CSS is not recommended, as the [base64 encoding](https://developer.mozilla.org/docs/Glossary/Base64) required to do so results in a larger payload and inlining it may well delay the discovery of other critical resources.
{% endAside %}

## Download

After discovering your fonts the browser needs to download them. The number of font files, their encoding, and their file size can significantly affect how quickly a font is downloaded and rendered by the browser.

### Self-host your font files

Fonts can be served through third-party services, such as [Google Fonts](https://fonts.google.com/), or can be self-hosted on your origin. When using a third-party service, your web page needs to open a connection to the provider's domain before it can start downloading those fonts.

This overhead can be reduced using the `preconnect` resource hint. By using `preconnect`, you can tell the browser to open a connection to the  cross-origin sooner than the browser ordinarily would:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">  
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

The above HTML hints the browser to establish a connection to `fonts.googleapis.com` and a [CORS](https://developer.mozilla.org/docs/Web/HTTP/CORS) connection to `fonts.gstatic.com`. Some font providers—such as Google Fonts—serve CSS and font resources from different origins.

You can eliminate the need for a third-party connection by self-hosting your fonts. In most cases, self-hosting fonts should be faster than downloading them from a cross-origin. If you plan on self-hosting your fonts, check that your site is using a [Content Delivery Network (CDN)](/content-delivery-networks/), [HTTP/2](/performance-http2/) or HTTP/3, and sets the correct caching headers for those fonts.

### Use WOFF2

[WOFF2](https://www.w3.org/TR/WOFF2/) enjoys [wide browser support](https://caniuse.com/woff2) and the best compression—up to 30% better than WOFF. The reduced file size leads to quicker download times. The WOFF2 format is often the only one you'll ever need—unless you have to support legacy browsers.

### Subset

Font files typically include a wide range of different [glyphs](https://en.wikipedia.org/wiki/Glyph), which are needed to cover the wide variety of characters used in different languages. If your page serves content in only one language—or uses a single alphabet—then you can reduce the size of your font files through subsetting. 

A font subset is a reduced set of the glyphs that were included in the original font file. For example, instead of serving all glyphs, your page might serve a specific subset for Latin characters. Depending on the subset, removing glyphs can significantly reduce the size of a font file.

Some font providers—such as Google Fonts—offer font subsets automatically through the use of a query string parameter. For example, the `https://fonts.googleapis.com/css?family=OpenSans&subset=latin` URL will serve a stylesheet with fonts that only use the Latin alphabet. If you've decided to self-host your fonts, you will need to generate and host those subsets yourself using tools such as [glyphanger](https://github.com/zachleat/glyphhanger) or [subfont](https://github.com/Munter/subfont).

## Font rendering

After the browser has discovered and downloaded a font, it can render it. By default, the browser will block the rendering of any text that uses a web font until the font is downloaded. You can adjust the browser's behavior and configure what to show until the font has fully loaded using the [`font-display` CSS property](https://developer.mozilla.org/docs/Web/CSS/@font-face/font-display).

### `block`

The default value for `font-display` is `block`. With `block`, the browser will block the rendering of any text that uses the specified font. Different browsers behave slightly differently. Chromium and Firefox block rendering for up to a maximum of 3 seconds before using a fallback font. Safari will block indefinitely until the font has loaded.

### `swap`

[`swap` is the most widely used `font-display` value](https://almanac.httparchive.org/en/2022/fonts#fig-13). `swap` will not block rendering and show the text immediately in a fallback font before swapping in the font specified in a stylesheet. This allows you to show your content immediately without waiting for the web font to download. The downside, however, is that using `swap` causes a layout shift if the fallback font and the font specified in your CSS vary greatly in terms of line height, kerning, and other font metrics.

### `fallback`

The `fallback` value for `font-display` is something of a compromise between `block` and `swap`. Unlike `swap`, it will block rendering of a font, but swap in fallback text only for a very short period of time. Unlike `block`, however, the blocking period is extremely short. This can work well on fast networks where, if the font is downloaded quickly, the typeface that is swapped in will be the web font. However, if networks are slow, the fallback text will be seen first after the blocking period, and then swapped out when the web font arrives.

### `optional`

`optional` is the most stringent `font-display` value, and will only use the web font if it downloads within 100 milliseconds. If the font takes longer than that, it will not be used on the page, and the browser will use the fallback font while the web font is downloaded in the background and placed in the browser cache. Subsequent page navigations will use the web font immediately, since it will be available in the browser cache. `font-display: optional` avoids the layout shift seen with `swap`, but some users will not see the web font if it arrives too late.

### Font demos

{% Glitch 'learn-performance-fonts' %}
