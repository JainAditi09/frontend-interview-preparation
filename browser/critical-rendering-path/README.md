# Critical Rendering Path (CRP) — Interview Preparation

**Priority:** P0/P1 · **Level:** SDE-1 to Senior · **Category:** Browser performance

> **Revision slogan:** Optimize what the browser must discover, download, parse and execute before it can render useful content.

## 1. What is the Critical Rendering Path?

The Critical Rendering Path is the sequence of steps and resource dependencies the browser goes through to render the initial page. It includes processing HTML, discovering and loading CSS, handling JavaScript that can block parsing or rendering, building the render tree, calculating layout and painting pixels.

The goal is to reduce the time until users see useful content and can interact with the page.

It is closely related to the browser rendering pipeline, but the focus differs:
- Rendering pipeline: how the browser computes and draws a frame.
- CRP: how resource discovery, download and processing affect the initial render.

## 2. Simplified CRP steps

1. Receive HTML bytes.
2. Parse HTML and build the DOM.
3. Discover external CSS, JavaScript, images, fonts and other resources.
4. Download and parse CSS to build the CSSOM.
5. Handle scripts according to their loading/execution attributes and dependencies.
6. Combine DOM and computed styles into render information.
7. Calculate layout.
8. Paint and composite the initial frame.

The browser uses speculative/preload scanning and other optimizations, so resource discovery is not always strictly sequential.

## 3. Why CSS can block rendering

CSS is generally render-blocking for the initial render because the browser needs relevant styles to avoid displaying unstyled content. A stylesheet can also affect when scripts execute: classic scripts may wait for stylesheets that are already being processed because script execution could inspect computed styles.

Example:

~~~html
<head>
  <link rel="stylesheet" href="/styles.css">
</head>
<body>
  <h1>Hello</h1>
</body>
~~~

The browser needs the stylesheet to compute the correct initial presentation. Large or unnecessary CSS can delay the first render.

Optimization:
- Remove unused CSS.
- Minify and compress stylesheets.
- Inline only small, truly critical CSS when it benefits the measured page.
- Load non-critical styles without blocking initial content, using a carefully tested strategy.
- Avoid importing large CSS bundles for a small initial viewport.

## 4. JavaScript and parser blocking

A classic script without async or defer can block HTML parsing while it is fetched and executed.

~~~html
<script src="/app.js"></script>
~~~

The parser pauses so the script can execute; this is necessary because the script may use document.write or depend on the DOM parsed so far.

### defer

~~~html
<script defer src="/app.js"></script>
~~~

- Downloads in parallel with HTML parsing.
- Executes after HTML parsing completes.
- Deferred classic scripts preserve document order.
- Executes before DOMContentLoaded.
- Does not block parsing while downloading.

### async

~~~html
<script async src="/analytics.js"></script>
~~~

- Downloads in parallel with HTML parsing.
- Executes as soon as it is ready, potentially pausing parsing.
- Execution order is not guaranteed across async scripts.
- Appropriate for independent scripts that do not depend on DOM parse completion or on one another.

### module

~~~html
<script type="module" src="/main.js"></script>
~~~

- Module scripts are deferred by default.
- They support imports and have module-scoped bindings.
- Their dependency graph is fetched and evaluated according to module loading rules.
- async can also be used with module scripts, changing when evaluation occurs.

### Script comparison

| Behaviour | Normal classic | defer | async | module |
|---|---|---|---|---|
| Downloads in parallel with parsing | No, parser pauses to fetch | Yes | Yes | Yes |
| Execution timing | Immediately when fetched | After parsing | As soon as ready | Deferred by default |
| Preserves order among same kind | Document order | Yes | No | Module graph semantics |
| DOM parsed before execution | Not guaranteed | Yes | Not guaranteed | Yes by default |
| Typical use | Small parser-dependent script | App code needing parsed DOM | Independent analytics | Modern modular app code |

## 5. Resource discovery and prioritization

The browser discovers resources through HTML parsing, CSS imports, scripts and other mechanisms. A resource that is discovered late can delay rendering even if its download is fast.

### Preload
Use rel=preload for a resource needed very soon but discovered late or otherwise not prioritized well.

~~~html
<link
  rel="preload"
  href="/fonts/brand.woff2"
  as="font"
  type="font/woff2"
  crossorigin
>
~~~

Preload should be used sparingly. Preloading too many resources competes for bandwidth and can delay more important assets.

### Preconnect

~~~html
<link rel="preconnect" href="https://fonts.example.com" crossorigin>
~~~

Preconnect warms up the connection to an important origin, including DNS/TCP/TLS work as applicable. Use it only for origins the page will actually need.

### Prefetch

~~~html
<link rel="prefetch" href="/next-page.js" as="script">
~~~

Prefetch hints that a resource may be needed for a future navigation. It is generally lower priority than resources needed for the current page and is browser-controlled.

### Lazy loading

~~~html
<img src="/below-fold.webp" loading="lazy" alt="Product">
~~~

Lazy loading defers offscreen resources that are not needed for the initial viewport. Do not lazy-load the likely Largest Contentful Paint (LCP) image; it should be discoverable and load early.

## 6. Images and fonts

### Images
- Serve appropriately sized responsive images using srcset and sizes.
- Use modern formats such as AVIF or WebP when appropriate.
- Compress images and specify width/height or aspect ratio to reduce layout shifts.
- Lazy-load offscreen images.
- Ensure the LCP image is not accidentally delayed by lazy loading or late CSS background discovery.

### Fonts
- Use WOFF2 where supported.
- Subset fonts and load only required weights/styles.
- Preload only critical fonts that are used immediately.
- Use font-display: swap or another appropriate strategy to control text visibility.
- Avoid excessive font variants that increase network cost.

## 7. Performance metrics related to CRP

| Metric | What it measures | Practical focus |
|---|---|---|
| FCP | First Contentful Paint | Get initial content rendered quickly |
| LCP | Largest Contentful Paint | Prioritize the main visible content/image |
| CLS | Cumulative Layout Shift | Reserve space for images, ads and dynamic content |
| INP | Interaction to Next Paint | Keep interactions and main-thread work responsive |
| TTFB | Time to First Byte | Improve server response and delivery path |

Current Core Web Vitals focus on LCP, INP and CLS. FCP and TTFB are useful diagnostic metrics but are not Core Web Vitals.

Common good thresholds at the 75th percentile:
- LCP: 2.5 seconds or less.
- INP: 200 milliseconds or less.
- CLS: 0.1 or less.

## 8. Practical optimization checklist

- [ ] Reduce server response time and use effective CDN delivery.
- [ ] Minify and compress HTML, CSS and JavaScript.
- [ ] Remove unused CSS and JavaScript.
- [ ] Split bundles and load route-specific code only when needed.
- [ ] Use defer or modules for scripts that do not need to block parsing.
- [ ] Keep critical CSS small and available early.
- [ ] Preload only truly critical resources.
- [ ] Preconnect only to important origins.
- [ ] Optimize and prioritize the LCP image.
- [ ] Lazy-load below-the-fold images and non-critical content.
- [ ] Use responsive images and appropriate dimensions.
- [ ] Optimize fonts and avoid unnecessary font weights.
- [ ] Avoid long main-thread tasks during startup.
- [ ] Measure real-user performance and test on slower devices/networks.

## 9. Interview questions and answers

**Q1. What is the Critical Rendering Path?**

It is the set of steps and resource dependencies the browser processes to render the initial page: HTML parsing, resource discovery, CSS processing, relevant JavaScript execution, render-tree construction, layout and paint. Optimization aims to get useful content on screen sooner.

**Q2. What is the difference between CRP and the rendering pipeline?**

The rendering pipeline explains how a frame is computed and drawn. CRP focuses on the resources and dependencies that affect the initial render, such as HTML, CSS, JavaScript, fonts and images.

**Q3. Why is CSS render-blocking?**

The browser needs applicable styles to render the page correctly and avoid a flash of unstyled content. Stylesheets can also affect script execution when scripts may depend on computed styles.

**Q4. Why can a normal script block HTML parsing?**

A classic script may inspect or modify the DOM and may use document.write. The parser pauses while the script is fetched and executed to preserve correct document semantics.

**Q5. What is the difference between async and defer?**

Both allow downloading in parallel with HTML parsing. Defer executes after parsing and preserves order among deferred classic scripts. Async executes as soon as it is ready and does not preserve order, making it suitable for independent scripts.

**Q6. Does defer guarantee the script runs after DOMContentLoaded?**

No. Deferred classic scripts execute after parsing but before the DOMContentLoaded event. The event waits for deferred scripts to execute.

**Q7. Are module scripts deferred by default?**

Yes. Module scripts are deferred by default. They can use imports and are evaluated according to module dependency rules.

**Q8. What is resource preload?**

Preload tells the browser to fetch a resource needed soon for the current page, such as a critical font or hero image. It should be used selectively because excessive preloads compete for bandwidth.

**Q9. What is preconnect?**

Preconnect asks the browser to establish an early connection to a required origin, reducing connection setup latency when the resource is later requested.

**Q10. What is prefetch?**

Prefetch hints that a resource may be needed in a future navigation. It is generally lower priority than current-page critical resources and is not guaranteed to be fetched.

**Q11. Why should we not lazy-load the LCP image?**

The LCP image is often the largest visible content element. Lazy loading can delay its discovery and download, worsening LCP. Make it discoverable early and consider an appropriate fetch priority.

**Q12. What is the difference between FCP and LCP?**

FCP measures when the first content is painted. LCP measures when the largest eligible content element in the viewport is rendered. LCP better reflects when the main page content appears.

**Q13. How can JavaScript affect LCP?**

Parser-blocking scripts, long main-thread tasks, delayed image discovery and client-side rendering dependencies can delay the largest contentful paint. Reduce blocking work and make the main content resource discoverable early.

**Q14. How do you reduce CLS?**

Reserve dimensions for images and embeds, avoid inserting content above existing content unexpectedly, use stable ad slots, and manage font loading to minimize layout shifts.

**Q15. How do you optimize font loading?**

Use efficient WOFF2 files, subset glyphs, load only needed weights, preload only the critical font when justified, and choose an appropriate font-display value.

**Q16. What is code splitting and how does it help CRP?**

Code splitting divides JavaScript into smaller chunks. The browser downloads only code needed for the current route or interaction, reducing initial transfer and parse/execute cost. Avoid splitting critical code so aggressively that it creates extra request waterfalls.

**Q17. How do you identify render-blocking resources?**

Use Lighthouse and Chrome DevTools Performance/Network panels. Inspect the critical request chain, render-blocking CSS and scripts, resource priorities, transfer sizes and main-thread work.

**Q18. What is a critical request chain?**

A dependency chain where one resource must be discovered or processed before another important resource can be requested or executed. Deep chains can delay rendering; flatten or shorten them where practical.

## 10. Scenario-based questions

**Scenario 1: The page shows a blank screen for several seconds. What would you investigate?**

Check TTFB, render-blocking CSS, parser-blocking scripts, critical request chains, client-side rendering dependencies and main-thread long tasks. Use Network and Performance tools to find the actual bottleneck before changing loading attributes.

**Scenario 2: A hero image is the LCP element but loads late. What would you do?**

Ensure it is present in initial HTML or otherwise discoverable early, avoid lazy loading it, use responsive image sources, compress and size it appropriately, and consider preload or high fetch priority if measurements support it.

**Scenario 3: Analytics slows page startup. How would you load it?**

If it is independent of initial rendering and DOM setup, use an async script or a controlled tag manager strategy. Ensure it does not create long main-thread tasks or compete with critical resources.

**Scenario 4: The page loads quickly on desktop but slowly on mobile. Why?**

Mobile devices may have slower CPUs, less memory and constrained networks. Large JavaScript parse/execute costs, oversized images, excessive requests and long main-thread tasks become more visible. Test with realistic mobile throttling and reduce work.

**Scenario 5: A font swap causes visible layout movement. How do you improve it?**

Use font subsets and appropriate fallbacks with similar metrics, tune font loading, and consider font metric overrides such as size-adjust, ascent-override and descent-override where suitable. Measure CLS.

## 11. Last-minute revision sheet

| Term | Remember |
|---|---|
| CRP | Steps and dependencies required for initial render |
| DOM | Built from HTML |
| CSSOM | Built from CSS |
| Render-blocking CSS | Needed for styled initial render |
| Parser-blocking script | Classic script can pause HTML parsing |
| defer | Download parallel; execute after parse in order |
| async | Download parallel; execute when ready, order not guaranteed |
| module | Deferred by default |
| preload | Fetch important current-page resource early |
| preconnect | Warm up connection to an important origin |
| prefetch | Hint for likely future navigation resource |
| lazy loading | Defer offscreen/non-critical resources |
| LCP | Largest visible content render time |
| INP | Interaction responsiveness |
| CLS | Visual stability |

### Explain in 30 seconds

"The Critical Rendering Path is the browser work and resource dependency chain required to render initial content. I optimize it by reducing server and resource latency, keeping critical CSS small, avoiding parser-blocking scripts, using defer or modules appropriately, prioritizing the LCP resource, lazy-loading below-the-fold assets and reducing main-thread JavaScript. I validate improvements using real-user metrics and DevTools."

## 12. Self-check
- [ ] Explain the Critical Rendering Path in sequence.
- [ ] Compare normal, async, defer and module scripts.
- [ ] Explain why CSS can block rendering.
- [ ] Explain preload, preconnect and prefetch.
- [ ] Explain why LCP images should not be lazy-loaded.
- [ ] Explain FCP, LCP, INP and CLS.
- [ ] Identify render-blocking resources in DevTools.
- [ ] Propose optimizations based on measured bottlenecks.

## 13. Further reading
- [web.dev: Critical rendering path](https://web.dev/articles/critical-rendering-path)
- [MDN: How browsers work](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work)
- [MDN: Script element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script)
- [web.dev: Optimize LCP](https://web.dev/articles/optimize-lcp)
- [web.dev: Web Vitals](https://web.dev/articles/vitals)

---
**Next recommended chapter:** Browser Storage.
