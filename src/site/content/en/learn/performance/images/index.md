---
title: Optimizing images
authors:
  - jlwagner
description: TODO
date: 2023-09-01
tags:
  - performance
---

{% Aside 'important' %}
The following is a general overview of image performance. For an in-depth treatment on the subject, check out the [Learn Images course](/learn/images/),
{% endAside %}

Images are the [heaviest](https://almanac.httparchive.org/en/2022/page-weight#fig-8) and [most prevalent](https://almanac.httparchive.org/en/2022/page-weight#fig-3) resource on the web. As a result, optimizing your images is expected to improve your page's performance. In most cases, optimizing your images means reducing the network time by sending fewer bytes.

An image is added to a page using [the `<img>` or `<picture>` elements](/learn/html/images/), or the CSS `background-image` property.

### Image size

The first optimization when it comes to using image resources is to display the image at the correct size. In the simplest case, if an image is displayed in a 500 pixel by 500 pixel container, using a square 1000 pixel image means that the image is twice as large as needed.

However, choosing which image size to use is complicated. In 2010, when the iPhone 4 was released, the screen resolution (640×960) was double that of the iPhone 3 (320×480). However, the screen size remained roughly the same. Running everything at the higher resolution would have made text and images tiny (half their previous size). Instead, 1 pixel became 2 [device pixels](https://en.wikipedia.org/wiki/Device-independent_pixel). This is called the [device pixel ratio (DPR)](https://developer.mozilla.org/docs/Web/API/Window/devicePixelRatio). The iPhone 4 had a DPR of 2.

Revisiting the earlier example, if the device has a DPR of 2 and the image is displayed in a 500 pixel by 500 pixel container, then a square 1000 pixel image (referred to as the [intrinsic size](https://developer.mozilla.org/docs/Glossary/Intrinsic_Size)) is now the optimal size. Similarly, if the device has a DPR of 3, then a square 1500 pixel image would be the optimal size.

{% Aside %}
In most cases, the [human eye may be unable to benefit from a DPR of 3](https://observablehq.com/@eeeps/visual-acuity-and-device-pixel-ratio) and you can probably use an image that is smaller than the optimal size without a noticeable decrease in image quality.
{% endAside %} 

#### `srcset`

The `<img>` element supports the [`srcset`](https://developer.mozilla.org/docs/Web/HTML/Element/img#attr-srcset) attribute, which allows you to specify a list of possible image sources the browser may use. Each image source specified must include the image URL, and a width _or_ pixel density descriptor.

```html
<img
  alt="An image"
  width="500"
  height="500"
  src="/image-500.jpg"
  srcset="/image-500.jpg 1x, /image-1000.jpg 2x, /image-1500.jpg 3x"
>
```

The above example uses the pixel density descriptor to hint the browser to use `image-500.png` on devices with a DPR of 1, `image-1000.jpg` on devices with a DPR of 2 and `image-1500.jpg` on devices with a DPR of 3.

#### `sizes`

The above solution only works if you display the image at the same CSS pixel size on all viewports. In many cases, the container's size will change depending on the user's device.

The [`sizes` attribute](https://developer.mozilla.org/docs/Web/HTML/Element/img#attr-sizes) allows you to indicate a list of source sizes, where each source size consists of a [media condition](https://developer.mozilla.org/docs/Web/CSS/Media_Queries/Using_media_queries#syntax) and a value. The `sizes` attribute describes the intended display size of the image in CSS pixels. When combined with the `srcset` width descriptors, the browser can choose which image source is best for the user's device.

```html
<img
  alt="An image"
  width="500"
  height="500"
  src="/image-500.jpg"
  srcset="/image-500.jpg 500w, /image-1000.jpg 1000w, /image-1500.jpg 1500w"
  sizes="(min-width: 768px) 500px, 100vw"
>
```

In the example above, the browser is instructed that if the device's viewport width exceeds 768 pixels, the image is displayed at a width of 500 pixels. On smaller devices, the image is displayed at 100vw—or the full viewport width.

The browser can then combine this information with the list of `srcset` image sources to find the optimal image. For example, if the user is on a mobile device with a screen width of 320 pixels with a DPR of 3, the image is displayed at  `320 CSS pixels × 3 DPR = 960 device pixels`. In this example, the closest sized image would be `image-1000.jpg` which has an intrinsic width of 1000 pixels.

{% Aside %}
`srcset` width descriptors do not work without the `sizes` attribute. Similarly, if you omit the `srcset` width descriptors, the `sizes` attribute won't do anything.
{% endAside %}

### File formats

Browsers support several different image file formats. Modern image formats like [WebP](/serve-images-webp/) and [AVIF](/compress-images-avif/) may provide better compression than PNG or JPEG, making your image file size smaller and therefore taking less time to download.

WebP is a [widely supported](https://caniuse.com/webp) format that works on all modern browsers. WebP often has better compression than JPEG, PNG, or GIF, offering both [lossy](https://en.wikipedia.org/wiki/Lossy_compression) and [lossless compression](https://en.wikipedia.org/wiki/Lossless_compression). WebP also supports alpha-channel transparency even when using lossy compression.

AVIF is a newer image format that is also [widely supported](https://caniuse.com/avif). AVIF supports both lossy and lossless compression, and [tests](https://netflixtechblog.com/avif-for-next-generation-image-coding-b1d75675fe4) have shown greater than 50% savings when compared to JPEG in some cases. AVIF also supports Wide Color Gamut (WCG) and High Dynamic Range (HDR).

### Compression

There are two types of compression:

1. [Lossy compression](https://en.wikipedia.org/wiki/Lossy_compression)
2. [Lossless compression](https://en.wikipedia.org/wiki/Lossless_compression).

Lossy compression works by reducing the image accuracy through [quantization](https://cs.stanford.edu/people/eroberts/courses/soco/projects/data-compression/lossy/jpeg/coeff.htm) while also reducing the color resolution through [chroma subsampling](https://en.wikipedia.org/wiki/Chroma_subsampling). Lossy compression is most effective on high-density images with a lot of noise and colors, but may be less effective with sharp edges, detail, or text. Lossy compression can be applied to JPEG, WebP, and AVIF images.

{% Aside %}
When using lossy compression, always confirm that the compressed image meets your quality standards. For example, images which contain high-contrast colored text atop a flat color are prone to artifacts from chroma subsampling.
{% endAside %}

Lossless compression reduces the file size by compressing an image with no data loss. Lossless compression describes a pixel based on the difference to its neighboring pixel(s). Lossless compression is used for the GIF, PNG, WebP, and AVIF image formats.

You can compress your images using [Squoosh](https://squoosh.app), [ImageOptim](https://imageoptim.com/), or an image optimization service. When compressing, there isn't a universal setting suitable for all cases. The recommended approach would be to experiment with different compression levels until you find a good compromise between image quality and file size.

### The `<picture>` element

The `<picture>` element gives you greater flexibility in specifying multiple image candidates:

```html
<picture>
  <source type="image/avif" srcset="image.avif">
  <source type="image/webp" srcset="image.webp">
  <img
    alt="An image"
    width="500"
    height="500"
    src="/image.jpg"
  >
</picture>
```

Using the `<source>` element within the `<picture>` element, you can add support for AVIF and WebP images, but fall back to JPEG if the browser does not support either. The browser will pick the first `<source>` element specified that matches. If it can render the image in that format, it will use that image. Otherwise, it will move on to the next `<source>`. In the example above, AVIF takes priority over WebP and WebP over JPEG.

A `<picture>` element requires an `<img>` element nested inside of it. The `alt`, `width`, and `height` attributes are defined on the `<img>` and used regardless of which `<source>` is selected.

The `<source>` element also supports the `media`, `srcset`, and `sizes` attributes. Similarly to the `<img>` example earlier, these indicate to the browser which image to select on different viewports.

{% Aside %}
While the `srcset` attribute provides the browser _hints_ on which image to use, the media query on the `<source>` element is a _command_ to be followed by the browser.
{% endAside %}

```html
<picture>
  <source
    media="(min-resolution: 1.5x)"
    srcset="/image-1000.jpg 1000w, /image-1500.jpg 1500w"
    sizes="(min-width: 768px) 500px, 100vw"
  >
  <img
    alt="An image"
    width="500"
    height="500"
    src="/image-500.jpg"
  >
</picture>
```

The `media` attribute takes a [media condition](https://developer.mozilla.org/docs/Web/CSS/Media_Queries/Using_media_queries#syntax). In the example above, the device's DPR is used as the media condition. Any device with a DPR greater than or equal to 1.5 would use the first `<source>` element. The `<source>` tells the browser that on devices with a viewport wider than 768 pixels, the selected image candidate is displayed at 500 pixels wide. On smaller devices, it will take up the entire viewport width. By combining the `media` and `srcset` attributes, you can have finer control over which image to use.

This is illustrated in the following table, where several viewport widths and device-pixel ratios are evaluated:

| Viewport Width (px) | 1 DPR | 1.5  DPR | 2  DPR |  3 DPR |
| ------------------- | -------:| --------:| --------:| --------:|
| 320                 | 500.jpg |  500.jpg |  500.jpg | 1000.jpg |
| 480                 | 500.jpg |  500.jpg | 1000.jpg | 1500.jpg |
| 560                 | 500.jpg | 1000.jpg | 1000.jpg | 1500.jpg |
| 1024                | 500.jpg | 1000.jpg | 1000.jpg | 1500.jpg |
| 1920                | 500.jpg | 1000.jpg | 1000.jpg | 1500.jpg |

Devices with a DPR of 1 download the `image-500.jpg` image, including most desktop users—who view the image at an extrinsic size of 500 pixels wide. On the other hand, mobile users with a DPR of 3 will download a potentially large `image-1500.jpg`—the same image used on desktop devices with a DPR of 3.

```html
<picture>
  <source
    media="(min-width: 560px) and (min-resolution: 1.5x)"
    srcset="/image-1000.jpg 1000w, /image-1500.jpg 1500w"
    sizes="(min-width: 768px) 500px, 100vw"
  >
  <source
    media="(max-width: 560px) and (min-resolution: 1.5x)"
    srcset="/image-1000-sm.jpg 1000w, /image-1500-sm.jpg 1500w"
    sizes="(min-width: 768px) 500px, 100vw"
  >
  <img
    alt="An image"
    width="500"
    height="500"
    src="/image-500.jpg"
  >
</picture>
```

In this example, the `<picture>` element is adjusted to include an additional `<source>` element to use different images for wide devices with a high DPR.

| Viewport Width (pixels) | 1 DPR |    1.5  DPR |    2  DPR |     3 DPR |
| ------------------- | -------:| -----------:| -----------:| -----------:|
| 320                 | 500.jpg |     500.jpg |     500.jpg | 1000-sm.jpg |
| 480                 | 500.jpg |     500.jpg | 1000-sm.jpg | 1500-sm.jpg |
| 560                 | 500.jpg | 1000-sm.jpg | 1000-sm.jpg | 1500-sm.jpg |
| 1024                | 500.jpg |    1000.jpg |    1000.jpg |    1500.jpg |
| 1920                | 500.jpg |    1000.jpg |    1000.jpg |    1500.jpg |

With this additional query, you can see that `image-1000-sm.jpg` and `image-1500-sm.jpg` are displayed on small viewports. This extra information allows you to compress your images further, since compression artifacts are not highly visible at that size and density, while also not compromising the image quality on desktop devices.

Alternatively, by adjusting the `srcset` and `media` attributes, you can avoid serving large images on small viewports:

```html
<picture>
  <source
    media="(min-width: 560px)"
    srcset="/image-500.jpg, /image-1000.jpg 2x, /image-1500.jpg 3x"
  >
  <source
    media="(max-width: 560px)"
    srcset="/image-500.jpg 1x, /image-1000.jpg 2x"
  >
  <img
    alt="An image"
    width="500"
    height="500"
    src="/image-500.jpg"
  >
</picture>
```

In the example above, the width descriptors have been removed in favor of device pixel ratio descriptors. Images served on a mobile device are limited to `/image-500.jpg` or `/image-1000.jpg`, even on devices with a DPR of 3.

### Managing complexity

When working with responsive images, you can find yourself with many different size variations and formats for each image. In the example above, variations for each size are used, but exclude AVIF and WebP. How many variants should you have? It depends.

While it may be tempting to have as many variations to deliver the best fit, every additional image variant comes at a cost and decreases the cache hit ratio. With only one variant, every user receives the same image, so it can be cached very efficiently. On the other hand, if there are many variations, each variant requires another cache entry. Server costs will increase and can degrade performance if the variant's cache entry has expired, and the image needs to be re-fetched from the origin server.

Apart from this, the size of your HTML document grows with each variation. You could find yourself shipping multiple kilobytes of HTML for each image.

#### `Accept` header

The [`Accept`](https://developer.mozilla.org/docs/Web/HTTP/Headers/Accept) HTTP request header informs the server which content types the user's browser understands. This information can be used by your server to serve the optimal image format without adding extra bytes to your HTML responses.

```js
if (request.headers.accept) {
  if (request.headers.accept.includes("image/avif")) {
    return reply.from(`image.avif`);
  } else if (request.headers.accept.includes("image/webp")) {
    return reply.from(`image.webp`);
  }
}

return reply.from(`image.jpg`);
```

The above snippet is a simplified version of the code you can add to your server's JavaScript backend to choose and serve the optimal image format. If the request `Accept` header includes `image/avif`, then it will serve the AVIF image. Otherwise, if the `Accept` header includes `image/webp` it will serve the WebP image. If neither of these conditions is true, then it will serve the JPG image.

This is not unlike behavior you would find on an [Image Content Delivery Network](/image-cdns/) (CDN). Image CDNs are excellent solutions for optimizing images and sending the optimal format based on the user's device and browser.

The key is to find a balance, generate a reasonable number of image candidates, and measure the impact on the user experience. Different images can give different results, and the optimizations applied to each image depend on its size within the page and the devices your users are using. For example, a full-width hero image may require more variants than thumbnail images on an e-commerce product listing page.

### Lazy loading

It is possible to tell the browser to lazy-load images when they appear in the viewport using the [`loading`](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/loading) attribute. A value of `lazy` tells the browser to not download the image until it is in (or near) the viewport. This saves bandwidth, allowing the browser to prioritize the resources it needs to render the critical content that is already in the viewport.

{% Aside 'important' %}
To learn more about lazy loading images, checkout [Deferring non-critical resources](Deferring%20non-critical%20resources.md)
{% endAside %}

### `decoding`

The [`decoding`](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/decoding) attribute tells the browser how it should decode the image. A value of `async` tells the browser that the image can be decoded asynchronously, possibly improving the time to render other content. A value of `sync` tells the browser that the image should be presented at the same time as other content. The default value of `auto` allows the browser to decide what is best for the user.

{% Aside %}
The effect of the `decoding` attribute may only be notable on larger, high-resolution images which take a much longer time to decode.
{% endAside %}

### Image demos

{% Glitch 'learn-performance-images' %}
