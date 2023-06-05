---
title: Demonstrating a Web Worker use case
authors:
  - jlwagner
description: 
date: 2023-09-01
tags:
  - performance
---

{% Aside 'codelab' %}
This section relies on a [Glitch demo](https://exif-worker.glitch.me/) to illustrate how a Web Worker can be used to offload work from the main thread onto a separate thread. Check it out and follow along!
{% endAside %}

Say you have a website that needs to strip [Exif metadata](https://en.wikipedia.org/wiki/Exif) from an image—this isn't such a far-fetched concept. In fact, websites like Flickr do this in order to provide data about an image, such as the color depth, camera make and model, and so on.

However, the logic for fetching an image, converting it to an [`ArrayBuffer`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer), and extracting the Exif data could be potentially expensive. Thankfully, the Web Worker scope allows for this work done on the main thread. Then, using the messaging pipeline, we can transmit the Exif data back to the main thread and construct it into HTML.

### What the main thread looks like without a Web Worker

First, let's observe what the main thread looks like when we do this work without a Web Worker:

1. Open a new tab in Chrome, and open its DevTools.
2. Open the **performance panel**.
3. Navigate to [https://exif-worker.glitch.me/without-worker.html](https://exif-worker.glitch.me/without-worker.html).
4. In the performance panel, click **Record** at the upper right hand corner of the DevTools pane.
5. Paste [this image link](https://upload.wikimedia.org/wikipedia/commons/5/59/Vespula_germanica_Richard_Bartz.jpg) (or another one of your choosing that contains Exif metadata) in the field and click the **Get that JPEG!** button.
6. Once the interface populates with Exif metadata, click **Record** again to stop recording.

<figure>
  {% Img src="image/jL3OLOhcWUQDnR4XjewLBx4e3PC3/fZWDAnNGIfzW37adIIIg.png", alt="The performance profiler showing the image metadata extractor app's activity occurring entirely on the main thread. The there are two substantial long tasks—one that runs a fetch to get the requested image and decode it, and another that extracts the metadata from the image.", width="800", height="198" %}
  <figcaption>
    Main thread activity in the image metadata extractor app. Note that all of the activity occurs on the main thread.
  </figcaption>
</figure>

You'll note that—aside from other threads that are always present, such as rasterizer threads and so on—everything in the app occurs on the main thread. On the main thread, the following things happen:

1. The form takes the input and dispatches a `fetch` request to get the initial portion of the image containing the Exif metadata.
2. The image data is converted into an `ArrayBuffer`.
3. The [`exif-reader`](https://www.npmjs.com/package/exif-reader) script is used to extract the Exif metadata from the image data.
4. The metadata is looped over to construct an HTML string, which then populates the metadata viewer.

Now let's contrast that with an implementation of the same behavior, but using a Web Worker.

### What the main thread looks like _with_ a Web Worker

Now that you've seen what things look like to parse a JPEG's Exif metadata on the main thread, let's take a look at what it looks like when a Web Worker is in the mix:

1. Open a another tab in Chrome, and open its DevTools.
2. Open the **performance panel**.
3. Navigate to [https://exif-worker.glitch.me/with-worker.html](https://exif-worker.glitch.me/with-worker.html).
4. In the performance panel, click the **record button** at the upper right hand corner of the DevTools pane.
5. Paste [this image link](https://upload.wikimedia.org/wikipedia/commons/5/59/Vespula_germanica_Richard_Bartz.jpg) in the field and click the **Get that JPEG!** button.
6. Once the interface populates with Exif metadata, click the **record button** again to stop recording.

<figure>
  {% Img src="image/jL3OLOhcWUQDnR4XjewLBx4e3PC3/Cni5MqYYcGgsmxpssFvO.png", alt="The performance profiler showing the image metadata extractor app's activity occurring on both the main thread and a web worker thread. While there are still long tasks on the main thread, they are substantially shorter, with the image fetching/decoding and metadata extraction occurring entirely on a web worker thread. The only main thread work involves passing data to and from the web worker.", width="800", height="299" %}
  <figcaption>
    Main thread activity in the image metadata extractor app. Note that there is an additional Web Worker thread where most of the work is done.
  </figcaption>
</figure>

This is the power of a Web Worker. Rather than doing _everything_ on the main thread, everything but populating the metadata viewer with HTML is done on a separate thread. This means that the main thread is freed up to do other work.

Perhaps the biggest advantage here is that, unlike the version of this app that doesn't use a Web Worker, the `exif-reader` script isn't loaded on the main thread, but rather on the Web Worker thread. This means that the cost of downloading, parsing, and compiling the `exif-reader` script takes place on the main thread.

Now let's dive into the Web Worker code that makes this all possible.

## A look at the Web Worker code

It's not enough to see the difference a Web Worker makes, you'll need to understand—at least in this case—what that code looks like so you'll understand what is possible in the Web Worker scope. Let's start with the main thread code that needs to occur before the Web Worker can enter the fray.

{% Aside %}
The code that follows isn't exhaustive, just the relevant parts. To see it all in action, [check out the Glitch demo in editor mode](https://glitch.com/edit/#!/exif-worker?path=js%2Fwith-worker%2Fscripts.js%3A1%3A10).
{% endAside %}

```js
// scripts.js

// Register the Exif reader Web Worker:
const exifWorker = new Worker('/js/with-worker/exif-worker.js');

// We have to send image requests through this proxy due to CORS limitations:
const imageFetchPrefix = 'https://res.cloudinary.com/demo/image/fetch/';

// Necessary elements we need to select:
const imageFetchPanel = document.getElementById('image-fetch');
const imageExifDataPanel = document.getElementById('image-exif-data');
const exifDataPanel = document.getElementById('exif-data');
const imageInput = document.getElementById('image-url');

// What to do when the form is submitted.
document.getElementById('image-form').addEventListener('submit', event => {
  // Don't let the form submit by default:
  event.preventDefault();

  // Send the image URL to the Web Worker on submit:
  exifWorker.postMessage(`${imageFetchPrefix}${imageInput.value}`);
});

// This listens for the Exif metadata to come back from the Web Worker:
exifWorker.addEventListener('message', ({ data }) => {
  // This populates the Exif metadata viewer:
  exifDataPanel.innerHTML = data.message;
  imageFetchPanel.style.display = 'none';
  imageExifDataPanel.style.display = 'block';
});
```

This code runs on the main thread, and sets up the form to send the image URL to the Web Worker. From there, the Web Worker code looks like this, starting with the messaging pipeline:

```js
// exif-worker.js

// Import the exif-reader script:
importScripts("/js/with-worker/exifreader.js");

// Set up a messaging pipeline to send the Exif data to the `window`:
self.addEventListener("message", ({ data }) => {
  getExifDataFromImage(data).then(status => {
    self.postMessage(status);
  });
});
```

This little bit of JavaScript sets up the messaging pipeline so that when the user submits the form with a URL to a JPEG image, the URL arrives in the Web Worker. From there, we use this bit of code to extract the Exif metadata from the raw image, build an HTML string, and send that HTML back to the `window`.

```js
// Takes a blob to transform the image data into an `ArrayBuffer`:
// NOTE: these promises are simplified for readability, and don't include
// rejections on failures. Check out the complete Web Worker code:
// https://glitch.com/edit/#!/exif-worker?path=js%2Fwith-worker%2Fexif-worker.js%3A10%3A5
const readBlobAsArrayBuffer = blob => new Promise(resolve => {
  const reader = new FileReader();

  reader.onload = () => {
    resolve(reader.result);
  };

  reader.readAsArrayBuffer(blob);
});

// Takes the Exif metadata and converts it to a markup string to
// display in the Exif metadata viewer in the DOM:
const exifToMarkup = exif => Object.entries(exif).map(([exifNode, exifData]) => {
  return `
    <details>
      <summary>
        <h2>${exifNode}</h2>
      </summary>
      <p>${exifNode === "base64" ? `<img src="data:image/jpeg;base64,${exifData}">` : typeof exifData.value === "undefined" ? exifData : exifData.description || exifData.value}</p>
    </details>
  `;
}).join("");

// Fetches a partial image and gets its Exif data
const getExifDataFromImage = imageUrl => new Promise(resolve => {
  fetch(imageUrl, {
    headers: {
      // We use a range request to avoid downloading the full image:
      "Range": `bytes=0-${(2 ** 10) * 64}`
    }
  }).then(response => {
    if (response.ok) {
      return response.clone().blob();
    }
  }).then(responseBlob => {
    readBlobAsArrayBuffer(responseBlob).then(arrayBuffer => {
      const tags = ExifReader.load(arrayBuffer, {
        expanded: true
      });

      resolve({
        status: true,
        message: Object.values(tags).map(tag => exifToMarkup(tag)).join("")
      });
    });
  });
});
```

It's a bit to read, but this is also a fairly complicated use case for Web Workers. However, the results are worth the work, and not just limited to this use case. You can use Web Workers for all sorts of things, such as isolating `fetch` calls and processing responses, processing large amounts of data without blocking the main thread, and that's just for starters.

When improving the performance of your web applications, start thinking about anything that can be done in a Web Worker context. The gains could be significant, and lead to an overall better user experience.

**TODO: ADD ASSESSMENT**
