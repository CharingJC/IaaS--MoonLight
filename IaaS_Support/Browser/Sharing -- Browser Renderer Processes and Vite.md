# Edge Chromium & Vite

### Browser multi-process architecture

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/265c190f-1779-4b82-b3b2-1f63fde08853/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/992b9863-0fb6-4a25-8bbf-bc5c63534613/Untitled.png)

### Inner workings of a Renderer Process

Renderer process touches many aspects of web performance.

The renderer process's core job is to turn HTML, CSS, and JavaScript into a web page that the user can interact with

![https://developers.google.com/web/updates/images/inside-browser/part3/renderer.png](https://developers.google.com/web/updates/images/inside-browser/part3/renderer.png)

Figure 1: Renderer process with a main thread, worker threads, a compositor thread, and a raster thread inside

Worker threads/Service worker is a way to write network proxy in your application code; allowing web developers to have more control over what to cache locally and when to get new data from the network. If service worker is set to load the page from the cache, there is no need to request the data from the network.

The important part to remember is that service worker is JavaScript code that runs in a renderer process. But when the navigation request comes in, how does a browser process know the site has a service worker?

![https://developers.google.com/web/updates/images/inside-browser/part2/scope_lookup.png](https://developers.google.com/web/updates/images/inside-browser/part2/scope_lookup.png)

Figure 10: the network thread in the browser process looking up service worker scope

When a service worker is registered, the scope of the service worker is kept as a reference (you can read more about scope in this [The Service Worker Lifecycle](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle) article). When a navigation happens, network thread checks the domain against registered service worker scopes, if a service worker is registered for that URL, the UI thread finds a renderer process in order to execute the service worker code. The service worker may load data from cache, eliminating the need to request data from the network, or it may request new resources from the network.

### Parsing

### Construction of a DOM

When the renderer process receives a commit message for a navigation and starts to receive HTML data, the main thread begins to parse the text string (HTML) and turn it into a **D**ocument **O**bject **M**odel (**DOM**).

### Subresource loading

If there are things like `<img>` or `<link>` in the HTML document, preload scanner peeks at tokens generated by HTML parser and sends requests to the network thread in the browser process.

### JavaScript can block the parsing

The HTML parser has to wait for JavaScript to run before it can resume parsing of the HTML document. Because JavaScript can change the shape of the document using things like `document.write()` which changes the entire DOM structure

![https://developers.google.com/web/updates/images/inside-browser/part3/dom.png](https://developers.google.com/web/updates/images/inside-browser/part3/dom.png)

Figure 2: The main thread parsing HTML and building a DOM tree

## Hint to browser how you want to load resources

There are many ways web developers can send hints to the browser in order to load resources nicely. If your JavaScript does not use `document.write()`, you can add `[async](<https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#attr-async>)` or `[defer](<https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#attr-defer>)` attribute to the `<script>` tag. The browser then loads and runs the JavaScript code asynchronously and does not block the parsing. You may also use [JavaScript module](https://developers.google.com/web/fundamentals/primers/modules) if that's suitable. `<link rel="preload">` is a way to inform browser that the resource is definitely needed for current navigation and you would like to download as soon as possible. You can read more on this at [Resource Prioritization – Getting the Browser to Help You](https://developers.google.com/web/fundamentals/performance/resource-prioritization).

### Updating rendering pipeline is costly

The most important thing to grasp in rendering pipeline is that at each step the result of the previous operation is used to create new data. For example, if something changes in the layout tree, then the Paint order needs to be regenerated for affected parts of the document.

If you are animating elements, the browser has to run these operations in between every frame. Most of our displays refresh the screen 60 times a second (60 fps); animation will appear smooth to human eyes when you are moving things across the screen at every frame. However, if the animation misses the frames in between, then the page will appear "janky".

![https://developers.google.com/web/updates/images/inside-browser/part3/pagejank1.png](https://developers.google.com/web/updates/images/inside-browser/part3/pagejank1.png)

Figure 11: Animation frames on a timeline

Even if your rendering operations are keeping up with screen refresh, these calculations are running on the main thread, which means it could be blocked when your application is running JavaScript.

![https://developers.google.com/web/updates/images/inside-browser/part3/pagejank2.png](https://developers.google.com/web/updates/images/inside-browser/part3/pagejank2.png)

Figure 12: Animation frames on a timeline, but one frame is blocked by JavaScript

You can divide JavaScript operation into small chunks and schedule to run at every frame using `requestAnimationFrame()`. For more on this topic, please see [Optimize JavaScript Execution](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution) . You might also run your [JavaScript in Web Workers](https://www.youtube.com/watch?v=X57mh8tKkgE) to avoid blocking the main thread.

![https://developers.google.com/web/updates/images/inside-browser/part3/raf.png](https://developers.google.com/web/updates/images/inside-browser/part3/raf.png)

Figure 13: Smaller chunks of JavaScript running on a timeline with animation frame

```
const element = document.getElementById('some-element-you-want-to-animate');
let start, previousTimeStamp;

function step(timestamp) {
  if (start === undefined)
    start = timestamp;
  const elapsed = timestamp - start;

  if (previousTimeStamp !== timestamp) {
    // Math.min() is used here to make sure the element stops at exactly 200px
    const count = Math.min(0.1 * elapsed, 200);
    element.style.transform = 'translateX(' + count + 'px)';
  }

  if (elapsed < 2000) { // Stop the animation after 2 seconds
    previousTimeStamp = timestamp
    window.requestAnimationFrame(step);
  }
}

window.requestAnimationFrame(step);
```

In this example, an element is animated for 2 seconds (2000 milliseconds). The element moves at a speed of 0.1px/ms to the right, so its relative position (in CSS pixels) can be calculated in function of the time elapsed since the start of the animation (in milliseconds) with 0.1 * elapsed. The element's final position is 200px (0.1 * 2000) to the right of its initial position.

![https://developers.google.com/web/updates/images/inside-browser/part3/composit.png](https://developers.google.com/web/updates/images/inside-browser/part3/composit.png)

Figure 18: Compositor thread creating compositing frame. Frame is sent to the browser process then to GPU

The benefit of compositing is that it is done without involving the main thread. Compositor thread does not need to wait on style calculation or JavaScript execution. This is why [compositing only animations](https://www.html5rocks.com/en/tutorials/speed/high-performance-animations/) are considered the best for smooth performance. If layout or paint needs to be calculated again then the main thread has to be involved.

## Wrap Up

In prev part, we looked at rendering pipeline from parsing to compositing.

In the following post, we'll look at how compositor is enabling smooth interaction when user input comes in.

## Input events from the browser's point of view

From the browser's point of view, input means any gesture from the user. Mouse wheel scroll is an input event and touch or mouse over is also an input event.

![https://developers.google.com/web/updates/images/inside-browser/part4/input.png](https://developers.google.com/web/updates/images/inside-browser/part4/input.png)

Figure 1: Input event routed through the browser process to the renderer process

When the compositor thread sends an input event to the main thread, the first thing to run is a hit test to find the event target. Hit test uses paint records data that was generated in the rendering process to find out what is underneath the point coordinates in which the event occurred.

## Minimizing event dispatches to the main thread

If a continuous event like `touchmove` was sent to the main thread 120 times a second, then it might trigger excessive amount of hit tests and JavaScript execution compared to how slow the screen can refresh.

![https://developers.google.com/web/updates/images/inside-browser/part4/rawevents.png](https://developers.google.com/web/updates/images/inside-browser/part4/rawevents.png)

Figure 7: Events flooding the frame timeline causing page jank

To minimize excessive calls to the main thread, Chrome coalesces continuous events (such as `wheel`, `mousewheel`, `mousemove`, `pointermove`, `touchmove` ) and delays dispatching until right before the next `requestAnimationFrame`.

![https://developers.google.com/web/updates/images/inside-browser/part4/coalescedevents.png](https://developers.google.com/web/updates/images/inside-browser/part4/coalescedevents.png)

Figure 8: Same timeline as before but event being coalesced and delayed

Any discrete events like `keydown`, `keyup`, `mouseup`, `mousedown`, `touchstart`, and `touchend` are dispatched immediately.

## Next steps

In this series, we've covered inner workings of a web browser. If you have never thought about why DevTools recommends adding `{passive: true}` on your event handler or why you might write `async` attribute in your script tag, I hope this series shed some light on why a browser needs those information to provide faster and smoother web experience.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f3b0faac-9228-40cc-80f1-6233caf23d17/Untitled.png)

In Chrome, we switched to using streaming as the main way to parse: now also regular synchronous scripts are parsed that way (not inline scripts though). We also stopped canceling task-based parsing if the main thread needs it, since that just unnecessarily duplicates any work already done.

For ES modules, this happens in three steps.

1.  Construction — find, download, and parse all of the files into module records.
2.  Instantiation —find boxes in memory to place all of the exported values in (but don’t fill them in with values yet). Then make both exports and imports point to those boxes in memory. This is called linking.
3.  Evaluation —run the code to fill in the boxes with the variables’ actual values.

Even though native ESM is now widely supported, shipping unbundled ESM in production is still inefficient (even with HTTP/2) due to the additional network round trips caused by nested imports. To get the optimal loading performance in production, it is still better to bundle your code with tree-shaking, lazy-loading and common chunk splitting (for better caching).

## Vite

### Pain Points

在浏览器支持 ES 模块之前，JavaScript 并没有提供的原生机制让开发者以模块化的方式进行开发。这也正是我们对 “打包” 这个概念熟悉的原因：使用工具抓取、处理并将我们的源码模块串联成可以在浏览器中运行的文件。因此会带来两个问题，缓慢启动和缓慢更新。

When cold-starting the dev server, a bundler-based build setup has to eagerly crawl and build your entire application before it can be served.

### **缓慢的服务器启动**

打包器的方式启动必须优先抓取并构建你的整个应用，然后才能提供服务。

![https://cn.vitejs.dev/assets/bundler.37740380.png](https://cn.vitejs.dev/assets/bundler.37740380.png)

Vite 以 [原生 ESM](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) 方式提供源码。这实际上是让浏览器接管了打包程序的部分工作，只需要在浏览器请求源码时进行转换并按需提供源码。

### **Slow Server Start**

Vite improves the dev server start time by first dividing the modules in an application into two categories: **dependencies** and **source code**.

-   **Dependencies** are mostly plain JavaScript that do not change often during development. Some large dependencies (e.g. component libraries with hundreds of modules) are also quite expensive to process. Dependencies may also be shipped in various module formats (e.g. ESM or CommonJS).
    
    Vite [pre-bundles dependencies](https://vitejs.dev/guide/dep-pre-bundling.html) using [esbuild](https://esbuild.github.io/). Esbuild is written in Go and pre-bundles dependencies 10-100x faster than JavaScript-based bundlers.
    
-   **Source code** often contains non-plain JavaScript that needs transforming (e.g. JSX, CSS or Vue/Svelte components), and will be edited very often. Also, not all source code needs to be loaded at the same time (e.g. with route-based code-splitting).
    
    Vite serves source code over [native ESM](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules). This is essentially letting the browser take over part of the job of a bundler: Vite only needs to transform and serve source code on demand, as the browser requests it. Code behind conditional dynamic imports is only processed if actually used on the current screen.
    

Vite 通过在一开始将应用中的模块区分为 **依赖** 和 **源码** 两类，改进了开发服务器启动时间。

-   **依赖** 大多为在开发时不会变动的纯 JavaScript。一些较大的依赖（例如有上百个模块的组件库）处理的代价也很高。依赖也通常会存在多种模块化格式（例如 ESM 或者 CommonJS）。
    
    Vite 将会使用 [esbuild](https://esbuild.github.io/) [预构建依赖](https://cn.vitejs.dev/guide/dep-pre-bundling.html)。Esbuild 使用 Go 编写，并且比以 JavaScript 编写的打包器预构建依赖快 10-100 倍。
    
-   **源码** 通常包含一些并非直接是 JavaScript 的文件，需要转换（例如 JSX，CSS 或者 Vue/Svelte 组件），时常会被编辑。同时，并不是所有的源码都需要同时被加载（例如基于路由拆分的代码模块）。
    
    ![https://cn.vitejs.dev/assets/esm.3070012d.png](https://cn.vitejs.dev/assets/esm.3070012d.png)
    

### **Slow Updates**

When a file is edited in a bundler-based build setup, it is inefficient to rebuild the whole bundle for obvious reasons: the update speed will degrade linearly with the size of the app.

Some bundler dev server runs the bundling in memory so that it only needs to invalidate part of its module graph when a file changes, but it still needs to re-construct the entire bundle and reload the web page. Reconstructing the bundle can be expensive, and reloading the page blows away the current state of the application. This is why some bundlers support Hot Module Replacement (HMR): allowing a module to "hot replace" itself without affecting the rest of the page. This greatly improves DX - however, in practice we've found that even HMR update speed deteriorates significantly as the size of the application grows.

In Vite, HMR is performed over native ESM. When a file is edited, Vite only needs to precisely invalidate the chain between the edited module and its closest HMR boundary (most of the time only the module itself), making HMR updates consistently fast regardless of the size of your application.

Vite also leverages HTTP headers to speed up full page reloads (again, let the browser do more work for us): source code module requests are made conditional via `304 Not Modified`, and dependency module requests are strongly cached via `Cache-Control: max-age=31536000,immutable` so they don't hit the server again once cached.

Once you experience how fast Vite is, we highly doubt you'd be willing to put up with bundled development again.

### **缓慢的更新**

基于打包器启动时，重建整个包的效率很低。原因显而易见：因为这样更新速度会随着应用体积增长而直线下降。

一些打包器的开发服务器将构建内容存入内存，这样它们只需要在文件更改时使模块图的一部分失活[[1]](https://cn.vitejs.dev/guide/why.html#footnote-1)，但它也仍需要整个重新构建并重载页面。这样代价很高，并且重新加载页面会消除应用的当前状态，所以打包器支持了动态模块热重载（HMR）：允许一个模块 “热替换” 它自己，而不会影响页面其余部分。这大大改进了开发体验 —— 然而，在实践中我们发现，即使采用了 HMR 模式，其热更新速度也会随着应用规模的增长而显著下降。

在 Vite 中，HMR 是在原生 ESM 上执行的。当编辑一个文件时，Vite 只需要精确地使已编辑的模块与其最近的 HMR 边界之间的链失活[[1]](https://cn.vitejs.dev/guide/why.html#footnote-1)（大多数时候只是模块本身），使得无论应用大小如何，HMR 始终能保持快速更新。

Vite 同时利用 HTTP 头来加速整个页面的重新加载（再次让浏览器为我们做更多事情）：源码模块的请求会根据 `304 Not Modified` 进行协商缓存，而依赖模块请求则会通过 `Cache-Control: max-age=31536000,immutable` 进行强缓存，因此一旦被缓存它们将不需要再次请求。