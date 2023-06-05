---
title: An overview of Web Workers
authors:
  - jlwagner
description: 
date: 2023-09-01
tags:
  - performance
---

{% Aside %}
This is a module that discusses how Web Workers work at a high level. It is not an exhaustive explanation, but rather an overview of Web Workers, and a [subsequent demonstration](/learn/performance/web-worker-demo/) of how they can improve performance. For a deep dive into the subject, read [Use web workers to run JavaScript off the browser's main thread](/off-main-thread/).
{% endAside %}

JavaScript is often described as a single-threaded language. When we call it the _main thread_, we mean that it's the only thread in which JavaScript is typically run. However, the truth is that there are ways to utilize additional threads for script-related tasks. One such feature in JavaScript that allows for multiple threads is known as [Web Workers](https://developer.mozilla.org/docs/Web/API/Web_Workers_API/Using_web_workers).

Web Workers are useful when you have computationally expensive work that just can't be run on the main thread without causing long tasks that make the page unresponsive—which can affect your website's [Interaction to Next Paint (INP)](/inp/). Or, you may want a way to reduce the amount of work already on the main thread so that other tasks have more room.

In this module, we'll learn about Web Workers, and show how you can use one to perform the computationally expensive work of reading image metadata off the main thread—and how you can get that information back _to_ the main thread.

## How a web worker is launched

A Web Worker is registered by instantiating the `Worker` class:

```javascript
const myNewWorker = new Worker('/js/my-new-worker.js');
```

In the worker JavaScript file—`my-new-worker.js` in this case—you can write code that then runs in its own thread.

## Web Worker limitations

Unlike JavaScript that runs on the main thread, Web Workers lack access to the `window` context and have limited access to the APIs it provides. Web Workers are subject to the following constraints and capabilities:

- Web Workers can't access the DOM—at least directly, anyway.
- Web Workers _can_ communicate with the `window` context through a messaging pipeline.
- The Web Worker scope is `self`, rather than `window`.
- Web Workers _do_ have access to JavaScript primitives and constructs, as well as APIs such as `fetch`—as well as [a limited set of others](https://developer.mozilla.org/docs/Web/API/Web_Workers_API#supported_web_apis).

## How Web Workers talk to the `window`

It's possible for Web Workers to talk to the `window` object through the use of a message pipeline. This pipeline allows you to ferry data to and from the `window` context to the Web Worker context. In your worker, you can wire up a `message` event that sends data to the `window`:

```js
self.addEventListener("message", () => {
  self.postMessage("Hello, window!");
});
```

Then in a script in the `window` context, you can receive the message from the worker using yet another `message` event:

```js
exifWorker.addEventListener("message", ({ data }) => {
  // Echoes "Hello, window!" to the console from the worker.
  console.log(data);
});
```

{% Aside %}
For a simpler way to use Web Workers—including a simpler messaging pipeline—consider using [Comlink](https://www.npmjs.com/package/comlink) when building Web Workers.
{% endAside%}

The messaging pipeline is an escape hatch from the Web Worker context, and allows you to ferry data to the `window` so that you can update the DOM—as well as other operations.

{% Assessment 'web-workers-overview' %}

In the next module, we'll demonstrate a concrete use case of a Web Worker, where a Web Worker will be used to read Exif metadata from an image off the main thread.
