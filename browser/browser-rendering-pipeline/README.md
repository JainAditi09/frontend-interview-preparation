# Browser Rendering Pipeline — Interview Preparation

**Priority:** P0/P1 · **Level:** SDE-1 to Senior · **Category:** Browser internals

> **Revision slogan:** DOM + CSSOM → render tree → layout → paint → compositing. This is a useful simplified model; browser engines may optimize and combine stages.

## 1. What is the browser rendering pipeline?

The rendering pipeline is the process through which the browser turns HTML, CSS and visual state into pixels on the screen. Understanding it helps explain why some UI updates are smooth while others cause layout shifts, dropped frames or jank.

A simplified pipeline:

1. Parse HTML and build the DOM.
2. Parse CSS and build the CSSOM.
3. Combine relevant DOM and style information into a render tree.
4. Calculate layout (geometry and positions).
5. Paint visual fragments into drawing instructions/layers.
6. Composite layers into the final frame.

Not every change triggers every stage. Browsers optimize work, and modern rendering engines may pipeline or parallelize parts of the process.

## 2. Step-by-step

### Step 1: DOM construction
The HTML parser tokenizes markup and builds a tree of nodes called the Document Object Model (DOM). JavaScript can inspect and modify the DOM.

### Step 2: CSSOM construction
The browser parses CSS into style rules and builds the CSS Object Model (CSSOM). Styles are matched and computed for elements.

### Step 3: Render tree
The browser determines which elements participate in rendering and combines structure with computed style. The render tree is not identical to the DOM: for example, elements with display:none do not generate render boxes, while pseudo-elements can generate rendered content without ordinary DOM nodes.

### Step 4: Layout (reflow)
The browser calculates the geometry of boxes: sizes, positions and relationships. Layout can be local or broader depending on the change and engine optimizations.

### Step 5: Paint
The browser creates drawing instructions for visual properties such as text, colors, borders, shadows and images.

### Step 6: Compositing
The compositor combines painted layers into the frame displayed on screen. Some animations can be handled efficiently by updating compositor properties without rerunning layout and paint for every frame.

## 3. What triggers which work?

| Change | Typical work | Why |
|---|---|---|
| Change element width/height | Layout + paint + composite | Geometry changes |
| Change margin/padding | Layout + paint + composite | Box geometry changes |
| Change font size | Layout + paint + composite | Text metrics and geometry change |
| Change background color | Paint + composite | Geometry usually unchanged |
| Change border color | Paint + composite | Visual pixels change |
| Change transform | Often composite only | Can be handled by compositor |
| Change opacity | Often composite only | Can be handled by compositor |
| Add/remove DOM node | Style recalculation, layout and paint as needed | Tree and geometry may change |

These are typical outcomes, not absolute rules. Exact invalidation depends on property, browser engine, containment, layer promotion and surrounding layout.

## 4. Reflow vs repaint vs composite

### Reflow (layout)
Reflow is the common term for recalculating layout after a change affects geometry. It can be expensive because layout dependencies may extend beyond the changed element.

Examples: changing width or height, changing font size, adding content that changes surrounding geometry. Reading layout after making style changes can force synchronous layout.

### Repaint
Repaint means redrawing visual appearance without necessarily recalculating geometry.

Examples: changing background color, text color or box-shadow.

### Composite
Compositing combines layers to produce the final frame. Properties such as transform and opacity can often be updated at this stage, avoiding layout and paint work.

### Interview one-liner
Layout calculates where and how big elements are; paint determines what they look like; compositing combines layers into the final frame.

## 5. Forced synchronous layout and layout thrashing

A forced synchronous layout can happen when JavaScript writes styles and then immediately reads layout-dependent values. The browser may need to flush pending style and layout work so it can return an up-to-date measurement.

Problematic pattern:

~~~js
const cards = document.querySelectorAll(".card");

cards.forEach((card) => {
  card.style.width = "200px";       // write
  console.log(card.offsetHeight);   // read may force layout
});
~~~

Better: batch reads, then batch writes.

~~~js
const cards = [...document.querySelectorAll(".card")];

// Read phase
const heights = cards.map((card) => card.offsetHeight);

// Write phase
cards.forEach((card, index) => {
  card.style.height = \`\${heights[index]}px\`;
});
~~~

This reduces repeated read/write interleaving. For complex interactions, schedule visual writes with requestAnimationFrame and measure performance in DevTools.

## 6. How to optimize rendering performance

- Batch DOM reads and writes; avoid layout thrashing.
- Prefer transform and opacity for animations when appropriate.
- Avoid unnecessary DOM updates and excessive DOM depth.
- Virtualize very large lists.
- Use CSS containment where it fits the component's layout/paint needs.
- Avoid animating properties that trigger expensive layout or paint.
- Debounce or throttle high-frequency event work when appropriate.
- Break up long JavaScript tasks and move CPU-heavy computation to Web Workers.
- Measure with browser Performance tools instead of assuming a property is cheap.

**Caution:** Do not add will-change everywhere. It can consume memory and create unnecessary layers. Use it only when measurements justify it.

## 7. Critical rendering path vs rendering pipeline

The rendering pipeline describes how the browser computes and draws a frame or update. The **Critical Rendering Path (CRP)** focuses on the resources and steps needed to render the initial page as quickly as possible, especially HTML, CSS, JavaScript and their loading/execution dependencies.

A simplified distinction:
- Rendering pipeline: DOM/style/layout/paint/composite for frames and updates.
- Critical Rendering Path: optimize discovery, loading, parsing and execution of resources required for the first meaningful render.

See [Critical Rendering Path](../critical-rendering-path/README.md).

## 8. Interview questions and answers

**Q1. Explain the browser rendering pipeline.**

The browser parses HTML into the DOM and CSS into the CSSOM, combines relevant structure and styles into a render tree, calculates layout, paints visual instructions and composites layers into a frame. This is a simplified model; engines optimize and skip stages when possible.

**Q2. What is the difference between DOM and render tree?**

The DOM represents document structure. The render tree represents content and boxes that participate in visual rendering. They are not identical: display:none elements do not generate render boxes, while pseudo-elements may create rendered content.

**Q3. What is CSSOM?**

The CSS Object Model is the browser's parsed representation of CSS rules and styles. The browser uses it with the DOM to compute styles for rendered elements.

**Q4. What is reflow?**

Reflow (layout) recalculates element geometry, such as size and position, after a change affects layout.

**Q5. What is repaint?**

Repaint redraws visual appearance when pixels change but geometry does not necessarily need recalculation.

**Q6. What is compositing?**

Compositing combines layers into the final displayed frame. Some changes, such as opacity and transform, can often be handled at this stage.

**Q7. Which CSS properties are generally better for animation?**

Transform and opacity are often good choices because browsers can frequently animate them on the compositor without layout. Always profile; layer creation and other effects can still have costs.

**Q8. Why can changing width be more expensive than changing transform?**

Changing width typically affects layout geometry and can require layout and paint. Transform often changes the visual position or scale of an already laid-out element and may be composited.

**Q9. What is layout thrashing?**

It is repeated interleaving of layout-affecting writes and layout reads, forcing the browser to recalculate layout multiple times. Batch reads and writes to reduce it.

**Q10. What causes forced synchronous layout?**

Reading geometry such as offsetWidth, offsetHeight or getBoundingClientRect() after pending style/layout changes can force the browser to synchronously update layout to return accurate values.

**Q11. Does every DOM change trigger reflow?**

No. A DOM change may only affect style or paint, or it may not affect visible rendering. The browser tracks invalidation and performs the work required by the change.

**Q12. Does display:none affect the render tree?**

An element with display:none does not generate a render box. Its descendants are also not rendered. Changing it to a displayed value can require style recalculation, layout and paint.

**Q13. What is a stacking context?**

A stacking context is a self-contained group of elements layered together in the stacking order. It can be created by properties such as positioned elements with a non-auto z-index, opacity below 1, transforms and several other CSS features.

**Q14. What is a compositing layer?**

A browser-managed surface that can be composited with other surfaces. Layerization is an implementation detail and not every DOM element maps to its own layer.

**Q15. How does requestAnimationFrame help?**

It schedules a callback for a rendering update, usually before paint. It is useful for synchronizing visual changes with the browser's frame cycle and avoiding arbitrary timer-based animation loops.

**Q16. How do you debug a slow rendering interaction?**

Record the interaction in DevTools Performance, inspect long tasks, style recalculation, layout, paint and compositing activity. Identify forced layouts and expensive selectors or DOM changes, then optimize and measure again.

**Q17. What is the relationship between JavaScript and rendering?**

JavaScript can modify DOM and styles, invalidating style, layout or paint. Long main-thread JavaScript can also delay input and rendering because the browser needs execution time to process updates.

**Q18. Does a Web Worker improve rendering directly?**

It does not render DOM or manipulate the page. It can move expensive computation off the main thread, leaving more time for input handling and rendering.

## 9. Scenario-based questions

**Scenario 1: An animation stutters while a large array is processed. What would you do?**

Profile the page to confirm main-thread blocking. Move CPU-heavy processing to a Dedicated Worker or split it into chunks that yield to the browser. Use transform/opacity for suitable animations and schedule visual changes with requestAnimationFrame.

**Scenario 2: A page repeatedly reads offsetHeight after changing styles. What is the issue?**

It may force synchronous layout and cause layout thrashing. Batch all required measurements before applying writes, or use a read/write scheduling strategy.

**Scenario 3: A developer adds will-change: transform to every card. Is that a good optimization?**

No. It can allocate unnecessary compositing resources and increase memory use. Apply it selectively only after measuring a real bottleneck.

**Scenario 4: Changing a button's background color causes a repaint. Does it cause reflow?**

Usually not, because geometry has not changed. It generally needs paint and compositing, although exact browser invalidation can vary.

## 10. Last-minute revision sheet

| Term | Meaning |
|---|---|
| DOM | Parsed document structure |
| CSSOM | Parsed CSS rules and style representation |
| Render tree | Boxes/content participating in rendering |
| Layout / reflow | Calculate geometry |
| Paint | Draw visual appearance |
| Composite | Combine layers into final frame |
| Layout thrashing | Repeated forced layout from read/write interleaving |
| Forced reflow | Synchronous layout flush caused by a measurement |
| requestAnimationFrame | Callback aligned with rendering update |
| Transform / opacity | Often compositor-friendly animation properties |

### Explain in 30 seconds

"The browser builds the DOM and CSSOM, determines the render tree, calculates layout, paints visual instructions and composites layers into the frame. A geometry change often triggers layout and paint, while a color change usually needs paint. Transform and opacity can often be handled by the compositor. I optimize by batching DOM reads and writes, avoiding long main-thread tasks and measuring with DevTools."

## 11. Self-check
- [ ] Explain DOM, CSSOM and render tree.
- [ ] Explain layout vs paint vs compositing.
- [ ] Identify common layout-triggering changes.
- [ ] Explain layout thrashing and how to avoid it.
- [ ] Explain why transform/opacity are often good for animation.
- [ ] Explain requestAnimationFrame.
- [ ] Debug a slow interaction using Performance tools.
- [ ] Distinguish rendering pipeline from Critical Rendering Path.

## 12. Further reading
- [MDN: Rendering performance](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Rendering)
- [web.dev: Rendering performance](https://web.dev/articles/rendering-performance)
- [web.dev: Avoid large, complex layouts and layout thrashing](https://web.dev/articles/avoid-large-complex-layouts-and-layout-thrashing)
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)
- [Chrome DevTools: Performance overview](https://developer.chrome.com/docs/devtools/performance/overview)

---
**Next recommended chapter:** [Critical Rendering Path](../critical-rendering-path/README.md)
