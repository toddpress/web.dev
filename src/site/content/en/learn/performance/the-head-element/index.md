---
title: The `<head>` element
authors:
  - imkevdev
description: TODO
date: 2023-09-01
tags:
  - performance
---

The `<head>` is the first part of an HTML document, preceding the `<body>`. It often includes metadata for the requested page, but it also includes references to critical resources such as CSS files, and sometimes JavaScript files, as well as various directives that can affect loading performance. Understanding the impact of the `<head>` tag on your page's load time is crucial if you want to render your page quickly.

The HTML parser processes markup sequentially, meaning that it is effectively parsed line-by-line. Therefore, what you put in a page's `<head>` element and the  order of what you put into it could have a significant impact on metrics such as [First Contentful Paint  (FCP)](/fcp/), and in the case of some directives, also [Largest Contentful Paint (LCP)](/lcp/).

## Render blocking

CSS is a [render-blocking](/critical-rendering-path-render-blocking-css/) resource, as it blocks the browser from rendering any content until the [CSS Object Model (CSSOM)](https://developer.mozilla.org/docs/Web/API/CSS_Object_Model) is constructed. The browser blocks rendering to prevent a [Flash of Unstyled Content (FOUC)](https://en.wikipedia.org/wiki/Flash_of_unstyled_content).

{% YouTube "pXfxXVpg4Qk" %}

In the preceding video, there is a brief FOUC where you can see the page without any styling. Subsequently, all styles are applied once the page's CSS has loaded.

## Parser blocking

A [parser-blocking](/critical-rendering-path-adding-interactivity-with-javascript/) resource interrupts the HTML parser, such as a `<script>` element without `async` or `defer` attributes. When the parser encounters a `<script>` element, the browser needs to evaluate and execute the script before proceeding with parsing the rest of the HTML. This is by design, as the JavaScript may modify or access the DOM.

```html
<script src="/script.js"></script>
```

When using external JavaScript files (without `async` or `defer`), the parser is blocked from when the file is discovered until it is downloaded, parsed, and executed. When using inline JavaScript, the parser is similarly blocked until the inline script is parsed and executed.

{% Aside %}
A parser-blocking `<script>` must also wait for any in-flight render-blocking CSS resources to arrive and be parsed before the browser can execute it. This is also by design, as a script may access styles declared in the render-blocking stylesheet (for example, by using `element.getComputedStyle()`).
{% endAside %}

## Preload scanner

The [preload scanner](/preload-scanner/) is a browser optimization in the form of a secondary HTML parser that scans the raw HTML response to find and speculatively fetch resources before the primary HTML parser would discover them. For example, the preload scanner would allow the browser to start downloading a resource specified in an `<img>` element even when the HTML parser is blocked while fetching resources such as CSS and JavaScript.

To take advantage of the preload scanner, critical resources should be included in your HTML markup. The following resource loading patterns will not be considered by the preload scanner:

- Images loaded in CSS via the `background-image` property.
- Dynamically-loaded scripts in the form of `<script>` element markup injected into the DOM via JavaScript or dynamic [`import()`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Operators/import).
- HTML rendered on the client using JavaScript.
- CSS `@import` declarations.

These resource loading patterns are all late-discovered resources, and therefore do not benefit from the preload scanner. Avoid them whenever possible. If avoiding such patterns _isn't_ possible, however, you may be able to use a `preload` hint to avoid resource discovery delays.

## CSS

CSS determines the presentation and layout of a page. As described earlier, CSS is a render-blocking resource, so optimizing your CSS could have a considerable impact on your page's load time.

### Minification

[Minifying CSS](/minify-css/) files reduces the file size of a CSS resource, making them quicker to download. This is accomplished primarily by removing content from a source CSS file such as spaces and other invisible characters, and outputting the result to a newly optimized file. Some advanced CSS minifiers may employ additional optimizations, such as coalescing redundant rules into multiple selectors:

```css
/* Unminified CSS: */

/* Heading 1 */
h1 {
  font-size: 2em;
  color: #000000;
}

/* Heading 2 */
h2 {
  font-size: 1.5em;
  color: #000000;
}
```

```css
/* Minified CSS: */
h1,h2{color:#000}h1{font-size:2em}h2{font-size:1.5em}
```

CSS minification is a low-risk optimization that could improve your First Contentful Paint. Tools such as [bundlers](https://bundlers.tooling.report/) can automatically perform this optimization for you.

### Unused CSS

Before rendering any content, the browser needs to download and parse all stylesheets. The time required to complete parsing also includes styles that are unused on the current page. If you are using a bundler that combines all CSS resources into a single file, your users are likely downloading more CSS rules than needed to render the page.

To discover unused CSS for the current page, use the [Coverage tool](https://developer.chrome.com/docs/devtools/css/reference/#coverage) in Chrome DevTools.

<figure>
  TODO: INSERT IMAGE WITH POSSIBLE CAPTION
</figure>

Removing [unused CSS](https://developer.chrome.com/en/docs/lighthouse/performance/unused-css-rules/) has a two-fold effect: in addition to reducing download time, you are optimizing [render tree](/critical-rendering-path-render-tree-construction/) construction as the browser needs to process fewer CSS rules.

{% Aside 'important' %}
Depending on the architecture of your website, it may not be possible to completely eliminate unused CSS—nor should you expect to. Focus on big wins: if you see a large part of a CSS file that is unused by the current page, it may either be used by a different page (which you can move to a different file altogether), or be deleted entirely.
{% endAside %}

### `@import`

You should avoid `@import` declarations in CSS:

```css
/* Don't do this! */
@import url('style.css');
```

Similarly to how the `<link>` element works in HTML, the `@import` declaration in CSS allows you to import an external CSS resource from within a stylesheet. The major difference between these two approaches is that the HTML `<link>` element is part of the HTML response, and therefore discovered much sooner than a CSS file downloaded by an `@import` declaration.

The reason for this is that in order for an `@import` declaration to be discovered, the CSS file that contains it must first be downloaded. This results in what is known as a _request chain_ which, in the case of CSS, delays how long it takes for a page to initially render. Another drawback is that stylesheets loaded using an `@import` declaration will not be caught by the preload scanner, and are a late-discovered render-blocking resource.

```html
<link rel="stylesheet" href="style.css">
```

In most cases, you can replace the `@import` by using a `<link rel="stylesheet">` tag. `<link>` allows the stylesheets to be downloaded in parallel and reduces overall load time.

{% Aside %}
If you need to use `@import`—such as for [cascade layers](https://developer.mozilla.org/docs/Learn/CSS/Building_blocks/Cascade_layers) or third-party stylesheets—you can mitigate the delay by using the `preload` directive for the imported stylesheet.

Additionally, CSS preprocessors such as SASS commonly use the `@import` syntax as part of a developer experience improvement that allows for separate and more modularized source files. However, when a CSS preprocessor encounters `@import` declarations, the referenced files will be bundled and written into a single resulting CSS file.
{% endAside %}

### Inline critical CSS

The time it takes to download CSS files can significantly increase a page's FCP. Inlining critical styles in the document `<head>` eliminates the network request and can improve load time when done correctly. The remaining CSS can be loaded [asynchronously](https://www.filamentgroup.com/lab/load-css-simpler/) or placed at the end of  the `<body>` element's contents.

{% Aside 'key-term' %}
Critical CSS refers to the styles required to render content that is visible within the initial viewport—sometimes referred to as "above the fold".
{% endAside %}

```html
<head>
  <title>Page Title</title>
  <!-- ... -->
  <style>h1,h2{color:#000}h1{font-size:2em}h2{font-size:1.5em}</style>
</head>
<body>
  <!-- ... -->
  <link rel="stylesheet" href="non-critical.css">
</body>
```

{% Aside 'important' %}
Extracting and maintaining critical styles can be hard. Which styles should be included? Which viewport should be targeted? Can it be automated? What happens if a user scrolls down before the non-critical CSS has loaded (they will experience a FOUC)?

These are all important questions that should be considered, as your website's architecture may make using critical CSS prohibitively difficult. However, the performance benefits can be worth the effort, so investigate whether critical CSS is a viable option for your website!
{% endAside %}

On the downside, inlining a large amount of CSS will add more to the initial HTML response, and may slow down initial navigation to a page.  It also means the CSS is not cached for subsequent pages that may use the same CSS. Test and measure your page's performance to make sure the trade-offs are worth it. 

### CSS demos

{% Glitch 'learn-performance-css' %}

## JavaScript

JavaScript drives most of the interactivity on the web, but it comes at a cost. Shipping too much JavaScript can make your web page slow to respond during page load, and may even cause responsiveness issues that slow down interactions—both of which can be frustrating for users.

### Render-blocking JavaScript

When loading `<script>` elements without a `defer` or `async` attribute, the browser blocks parsing and rendering until the script is downloaded, parsed, and executed. Similarly, inline scripts block the parser until the script is parsed and executed.

### `async` vs `defer`

`async` and `defer` allow external scripts to load without blocking the HTML parser while scripts (including inline scripts) with `type="module"` are deferred automatically. However, `async` and `defer` have some important differences.

<figure>
  TODO: INSERT IMAGE
  <figcaption>
    Sourced from <a href="https://html.spec.whatwg.org/multipage/scripting.html" rel="noopener">https://html.spec.whatwg.org/multipage/scripting.html</a>
  </figcaption>
</figure>

Scripts loaded with `async` are parsed and executed immediately once downloaded, while scripts loaded with `defer` are executed when HTML document parsing is finished (at the time of the browser's `DOMContentLoaded` event). Additionally, `async` scripts may execute out-of-order, while `defer` scripts are executed in the order in which they appear in the markup.

{% Aside 'important' %}
Scripts loaded using `type="module"` are deferred by default, while scripts injected into the DOM via JavaScript behave like `async` scripts.
{% endAside %}

### Client-side rendering

Generally, you should avoid using JavaScript to render any critical content or a page's [LCP element](/lcp/#what-elements-are-considered). This is known as client-side rendering, and is a technique used extensively in Single Page Applications (SPAs).

Markup rendered by JavaScript sidesteps the preload scanner, as the resources contained within the client-rendered markup [are not discoverable](/preload-scanner/#rendering-markup-with-client-side-javascript) by it. This could delay the download of important resources, such as an LCP image. The browser will only begin downloading the LCP image after the script has executed and added the element to the DOM. In turn, the script can only be executed after it has been discovered, downloaded, and parsed. This is known as a [critical request chain](https://developer.chrome.com/en/docs/lighthouse/performance/critical-request-chains/) and should be avoided.

Additionally, rendering markup using JavaScript is more likely to generate [long tasks](/long-tasks-devtools/) than markup downloaded from the server in response to a navigation request. Extensive use of client-side rendering of HTML can negatively affect interaction latency. This is especially true in cases where [a page's DOM is very large](/dom-size-and-interactivity/), which triggers significant rendering work

### Minification

Similar to CSS, [minifying your JavaScript](https://developer.chrome.com/en/docs/lighthouse/performance/unminified-javascript/) can reduce file size. This can lead to quicker downloads, allowing the browser to move onto the process of parsing and compiling JavaScript more quickly.

### JavaScript demos

{% Glitch 'learn-performance-javascript' %}
