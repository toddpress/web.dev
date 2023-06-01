---
title: Video
authors:
  - imkevdev
description: TODO
date: 2023-09-01
tags:
  - performance
---

Videos are a popular media type used often on web pages. Before looking at some of the possible optimizations, let's look at how the `<video>` element works.

## Video source files

When working with media files, the file you recognize on your operating system (`.mp4`, `.webm`, and so on) is called a _container_. A container contains one or more streams. In most cases, this would be the video and audio stream. You can compress each stream using a codec. For example, a `video.webm` could be a WebM container containing a video stream compressed using [_VP9_](https://en.wikipedia.org/wiki/VP9) and an audio stream compressed using [_Vorbis_](https://en.wikipedia.org/wiki/Vorbis). Understanding the difference between containers & codecs is helpful because it enables you to compress your media files using significantly less bandwidth.

One method to compress your videos involves using [FFmpeg](https://ffmpeg.org/):

```bash
$ ffmpeg -i input.mov output.webm
```

The above command takes the `input.mov` file and outputs an `output.webm` file using the default FFmpeg options. These will give you a smaller file that works in most modern browsers. Tweaking the FFmpeg options could help you reduce the file size even further. For example, if you are using a `<video>` element to replace a GIF, you should remove the audio track:

```bash
$ ffmpeg -i input.mov -an output.webm
```

{% Aside 'important' %}
The [`-an`](https://ffmpeg.org/ffmpeg.html#Audio-Options) flag removes all audio streams from the output file. This is particularly important optimization if the use case for your video doesn't require audio (for example, where videos are used to replace animated GIFs), as removing the audio stream reduces the size of the video file.
{% endAside %}

### Multiple formats

When working with video files, specifying multiple formats works as a fallback for browsers that do not support all modern formats.

```html
<video>
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
</video>
```

Since [all modern browsers support the H.264 codec](https://caniuse.com/mpeg4), MP4 can be used as the fallback for legacy browsers. The WebM version can use the newer [AV1](https://en.wikipedia.org/wiki/AV1) codec, which is still [not widely supported](https://caniuse.com/av1), or the older VP9 codec, which is [better supported](https://caniuse.com/webm), but has worse compression than AV1.

{% Aside %}
Similar to the `<picture>` element, the order in which you list the `<source>` child elements in the `<video>` element dictates the priority to the browser. If you specify an MP4 source first, then the browser will select that format regardless of its support for more efficient alternative formats.
{% endAside %}

## The `poster` attribute

A video's poster image is added via the [`poster` attribute](https://developer.mozilla.org/docs/Web/HTML/Element/video#attr-poster), and can indicate to the viewer what the video content is about before starting playback. It also serves as a fallback in case the video fails to load.

```html
<video poster="poster.jpg">
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
</video>
```

{% Aside %}
A `<video>` element without a `poster` image is not an [LCP candidate](/lcp/#what-elements-are-considered)—there is an [open issue](https://bugs.chromium.org/p/chromium/issues/detail?id=1289664) to use the first frame once it has been painted. Currently, when there is no `poster` image, the `<video>` element is not considered an LCP candidate, and the next largest qualifying element is used.
{% endAside %}

## Autoplay

According to the HTTP Archive, [20% of videos](https://almanac.httparchive.org/en/2022/media#fig-37) across the web include the `autoplay` attribute. `autoplay` is used when a video must be played immediately—such as when used as a video background or a [replacement for animated GIFs](/replace-gifs-with-videos/). Animated GIFs can grow to several megabytes of data. You should generally avoid them, as `<video>` is a much more efficient format for this type of media

You can use the `<video>` element with the `autoplay` attribute:

```html
<video autoplay muted loop playsinline>
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
</video>
```

{% Aside 'caution' %}
A video with the `autoplay` attribute will begin downloading immediately, even if it is outside of the initial viewport.
{% endAside %}

By combining the `poster` attribute with the [Intersection Observer API](https://developer.mozilla.org/docs/Web/API/Intersection_Observer_API) you can configure your page to only [download videos once they are within the viewport](/lazy-loading-video/#video-gif-replacement). The `poster` image could be a low-quality image placeholder, such as the first frame of the video. Once the video appears in the viewport, the browser will begin downloading the video. This could improve load times for content within the initial viewport. On the downside, when using a `poster` image for `autoplay`, your users will download an image that is shown briefly until the video has loaded. 

{% Aside %}
Autoplaying video on page load can be an unpleasant experience for users—especially if the video contains an audio stream. Autoplay should only be used when necessary, and as the result of an anticipated need for the user. Browsers have different criteria that make a video eligible to autoplay. For more details, see the autoplay policies for [Chrome](https://developer.chrome.com/blog/autoplay/) and [WebKit](https://webkit.org/blog/7734/auto-play-policy-changes-for-macos/). 
{% endAside %}

## User-initiated playback

Generally, the browser begins downloading a video file as soon as the HTML parser discovers the `<video>` element. If you have `<video>` elements where the user initiates playback, you probably don't want the video to begin downloading until the user has interacted with it.

You can delay downloading by using the `<video>` element's [`preload`](https://developer.mozilla.org/docs/Web/HTML/Element/video#attr-preload) attribute. Setting the value to `none` will inform the browser that the video should not be preloaded. Setting `preload="metadata"` will only fetch video metadata, such as the video's duration. Setting `preload="none"` will fetch nothing in the video in advance, which may be desirable if you're using the `poster` attribute.

{% Aside %}
The `preload` attribute is a _hint_—the browser may opt not to obey it, and it may behave differently on different browsers or when comparing a mobile device to a desktop device.
{% endAside %} 

You can improve the user experience by adding a `poster` image. This will help the user understand what the video is about. If the poster image is your LCP element, you can increase the `poster` image's priority using the `<link rel="preload">` hint along with `fetchpriority="high"`.

```html
<link rel="preload" as="image" href="poster.jpg" fetchpriority="high">
```

{% Aside %}
If the video is not the largest element in the viewport, preloading the `poster` image may delay your LCP through bandwidth contention, where available bandwidth would otherwise be allocated to other more critical resources.
{% endAside %}

## Embeds

Third-party video embeds can significantly impact page performance. According to the HTTP Archive, YouTube embeds block the main thread for more than [1.7 seconds](https://almanac.httparchive.org/en/2022/third-parties#fig-8) for the median website. This is true even if the user doesn't interact with the video.

Blocking the main thread is a serious user experience problem that can impact a page's [Interaction to Next Paint (INP)](/inp/) metric. Instead of loading the embed immediately during page load, you can use a facade. 

{% Aside 'important' %}
To learn more about using facades, read the [Deferring non-critical resources](TODO) module in this course.
{% endAside %}

## Video demos

{% Glitch 'learn-performance-video' %}
