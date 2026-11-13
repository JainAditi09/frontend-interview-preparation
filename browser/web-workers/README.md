# Web Workers — Interview Preparation

**Priority:** P0/P1 · **Level:** SDE-1 to Senior · **Category:** Browser internals

> **Revision slogan:** Dedicated = one owner; Shared = shared connections; Service = event-driven network/offline proxy.

## 1. What problem do Web Workers solve?

JavaScript running in a page normally executes on the browser's main thread. That thread coordinates JavaScript execution, user input and much of the rendering work. A long-running synchronous calculation can keep the main thread busy, delaying clicks, scrolling and paints.

A Web Worker lets an application run JavaScript in a separate worker execution context. The main thread and worker communicate by message passing. This is useful for CPU-heavy work such as parsing a large dataset, processing an image or performing expensive calculations.

**Important:** Workers do not make one JavaScript function magically faster. They move work off the main thread so the page can remain responsive. Worker startup, messaging and memory have costs.

## 2. The three worker types

### A. Dedicated Worker

A Dedicated Worker is associated with the creating owner (typically one document or another worker). It is not a shared endpoint for unrelated tabs.

**Use it for:** CPU-intensive calculations, parsing a large file, image transforms, compression and other work that should not block the UI.

Main thread — `main.js`:

```js
const worker = new Worker(
  new URL("./worker.js", import.meta.url),
  { type: "module" }
);

worker.addEventListener("message", (event) => {
  console.log("Result:", event.data);
});

worker.addEventListener("error", (event) => {
  console.error("Worker failed:", event.message);
});

worker.postMessage({ type: "DOUBLE", value: 21 });

// When no longer needed:
worker.terminate();
```

Worker — `worker.js`:

```js
self.addEventListener("message", (event) => {
  const { type, value } = event.data;

  if (type === "DOUBLE") {
    self.postMessage(value * 2);
  }
});
```

Notes:
- `new Worker(url)` creates a dedicated worker.
- `postMessage()` sends data to the worker; the worker replies with `self.postMessage()`.
- `worker.terminate()` stops it immediately from the owner side; alternatively the worker can call `self.close()`.
- Module workers can use standard module imports. Classic workers can use `importScripts()`.

### B. Shared Worker

A Shared Worker can be connected to by multiple same-origin browsing contexts, such as tabs, windows or iframes, when they resolve to the same shared worker. Each connection communicates through a `MessagePort`.

**Use it for:** coordinating state or communication among tabs, sharing one worker-managed resource, or avoiding duplicate work across same-origin contexts.

Shared worker — `shared-worker.js`:

```js
self.addEventListener("connect", (event) => {
  const port = event.ports[0];

  port.addEventListener("message", (messageEvent) => {
    port.postMessage({
      from: "shared-worker",
      received: messageEvent.data,
    });
  });

  port.start();
});
```

Connect from each tab:

```js
const shared = new SharedWorker(
  new URL("./shared-worker.js", import.meta.url),
  { type: "module", name: "app-shared-worker" }
);

shared.port.addEventListener("message", (event) => {
  console.log(event.data);
});

shared.port.start();
shared.port.postMessage("Hello from this tab");
```

Notes:
- The `connect` event exposes the connecting port through `event.ports[0]`.
- Communicate through `shared.port`, not the Dedicated Worker-style worker message handler.
- A Shared Worker is not durable background execution; its connections and lifetime are managed by the browser.
- Check browser support for the target audience.

### C. Service Worker

A Service Worker is an event-driven worker associated with a registration and scope. It can act as a programmable network intermediary for pages under that scope, enabling offline caching and capabilities such as push events.

**Use it for:** PWA offline behaviour, request interception, cache strategies, push notifications and supported background events.

Register it from the page:

```js
if ("serviceWorker" in navigator) {
  navigator.serviceWorker
    .register("/service-worker.js")
    .then((registration) => {
      console.log("Service Worker scope:", registration.scope);
    })
    .catch((error) => {
      console.error("Service Worker registration failed:", error);
    });
}
```

A minimal cache-first example — `service-worker.js`:

```js
const CACHE_NAME = "app-shell-v1";
const APP_SHELL = ["/", "/index.html", "/offline.html"];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(APP_SHELL))
  );
  self.skipWaiting();
});

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener("fetch", (event) => {
  if (event.request.method !== "GET") return;

  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      if (cachedResponse) return cachedResponse;

      return fetch(event.request).catch(async () => {
        if (event.request.mode === "navigate") {
          return (await caches.match("/offline.html")) ||
            new Response("You are offline", {
              status: 503,
              headers: { "Content-Type": "text/plain" },
            });
        }
        throw new Error("Network unavailable and resource not cached");
      });
    })
  );
});
```

Notes:
- Service Workers require a secure context (HTTPS); `localhost` is treated as secure for development.
- They are browser-managed and may be stopped when idle, then restarted for an event. They are not permanently running background threads.
- They have no direct DOM access.
- Registration does not automatically cache resources. The app must explicitly implement caching.
- Production applications need cache versioning, invalidation, failure handling and update UX.
- `skipWaiting()` and `clients.claim()` affect update/control timing; use them deliberately because a new worker version can take control while old pages are open.

## 3. Side-by-side comparison

| Feature | Dedicated Worker | Shared Worker | Service Worker |
|---|---|---|---|
| Main purpose | Off-main-thread work | Share endpoint across contexts | Network interception and background web capabilities |
| Consumers | One owner | Multiple same-origin contexts | Clients within registration scope |
| Creation/access | `new Worker()` | `new SharedWorker()` | `navigator.serviceWorker.register()` |
| Communication | `postMessage` | Per-connection `MessagePort` | `postMessage`, clients and lifecycle events |
| DOM access | No | No | No |
| Lifecycle | Associated with owner | Browser-managed connections | Browser-managed, event-driven |
| Typical events | `message`, `error` | `connect`, port messages | `install`, `activate`, `fetch`, `message`, `push` |
| Intercept page fetches? | No | No | Yes, within scope |
| Offline caching | Not primary role | Not primary role | Core use case with Cache Storage |
| Typical use | Heavy computation | Cross-tab coordination | PWA, offline and network handling |

### The 20-second interview answer

A Dedicated Worker belongs to one owner and is best for moving CPU-heavy work off the UI thread. A Shared Worker can be connected to by multiple same-origin tabs or frames through separate MessagePorts. A Service Worker is event-driven and browser-managed; it can intercept requests within its scope, enabling offline caching and PWA features.

## 4. Concepts interviewers expect

### Main thread vs worker thread
- The page main thread owns the DOM and UI work.
- A worker has a separate global execution context.
- A worker cannot directly read or modify the page DOM.
- A worker can use APIs available in its worker context; availability varies by API and browser.

### `self` vs `window`
- `window` is the page's Window global and is not available inside workers.
- `self` refers to the worker global scope inside a worker.
- In a Service Worker, `self` is a `ServiceWorkerGlobalScope`.

### Message passing and structured clone
`postMessage()` normally uses the structured clone algorithm. Many data types can be cloned, but functions and DOM nodes cannot. Large payloads can make copying expensive.

### Transferable objects
Transferables move ownership instead of copying the underlying resource. An `ArrayBuffer` is detached from the sender after a successful transfer.

```js
const buffer = new ArrayBuffer(1024);
worker.postMessage({ buffer }, [buffer]);
console.log(buffer.byteLength); // 0 after transfer
```

### Shared memory
`SharedArrayBuffer` can provide shared memory between agents. In browsers it generally requires cross-origin isolation (commonly configured with COOP and COEP response headers). Synchronization must be handled carefully, often using `Atomics`.

### Storage
Workers cannot use `localStorage`, but IndexedDB is available in worker contexts. Service Workers commonly use Cache Storage for Request/Response caching.

## 5. Interview questions and answers

### Fundamentals

**Q1. What are Web Workers and why do we need them?**

Web Workers let JavaScript run in a background execution context separate from the page's main thread. They are useful for CPU-intensive or long-running work so input handling and rendering are not blocked. Communication is asynchronous through messages.

**Q2. Do Web Workers create true parallelism?**

They execute in separate agents and can run in parallel on available CPU cores, subject to browser scheduling and hardware. A worker is not a guarantee of a dedicated physical core or a performance improvement for every task.

**Q3. Can a Web Worker access or manipulate the DOM? Why?**

No. Workers do not have access to `window`, `document` or page DOM nodes. They run in a separate execution context; DOM work remains on the main thread. Send data to the main thread and let the UI update there.

**Q4. What is the difference between `window` and `self`?**

`window` is the global object for a browser page. Worker code uses `self`, which refers to that worker's global scope. `self` is also available in the page, where it refers to the window.

**Q5. How does communication happen between the page and a worker?**

Through asynchronous message passing using `postMessage()` and message events. The payload is normally structured-cloned; transferables can be supplied to transfer ownership efficiently.

**Q6. What is the difference between Dedicated and Shared Workers?**

A Dedicated Worker is associated with one owner. A Shared Worker can accept connections from multiple same-origin browsing contexts. Shared Workers use a separate MessagePort per connection.

**Q7. What makes a Service Worker different from the other two?**

It is tied to a registration and scope rather than one page's computation. It is event-driven and can intercept scoped network requests, use Cache Storage and handle supported events such as push. The browser may stop it while idle and start it again later.

**Q8. Can a Service Worker run after the tab is closed?**

It can be started by the browser for supported events, such as a push event, even when no page is currently open. It is not a continuously running process; capabilities depend on browser/platform support and permissions.

**Q9. Can workers access browser storage?**

Workers cannot access `localStorage`, which is tied to the Window context. IndexedDB is available in workers. Service Workers can also use Cache Storage for Request/Response caching.

**Q10. Do Web Workers share memory with the main thread?**

Usually, messages are structured-cloned, so the contexts do not share ordinary object references. Transferable objects move ownership. `SharedArrayBuffer` is an advanced shared-memory option with browser security requirements and synchronization concerns.

### Behaviour and lifecycle

**Q11. What happens if a worker throws an uncaught error?**

The owning context can listen for the worker's `error` event. Production code should log useful context, handle expected failures, terminate or recreate the worker if appropriate, and keep the UI recoverable.

**Q12. How do you stop a Dedicated Worker?**

The owner can call `worker.terminate()` for immediate termination. The worker can also call `self.close()` to close itself.

**Q13. What are the Service Worker lifecycle stages?**

A registration is created; the worker installs; when eligible to take control, it activates; then it handles events such as fetch and push. During updates, an installing worker may wait until existing controlled clients are gone unless the lifecycle is deliberately changed.

**Q14. Why does a Service Worker require HTTPS?**

It has powerful capabilities, including request interception, so browsers restrict it to secure contexts. Localhost is treated as secure for development.

**Q15. What is the Service Worker scope?**

The scope determines which clients and requests the Service Worker can control. By default it is based on the directory containing the worker script. Script location and optional registration scope must follow browser scope rules.

**Q16. Does registering a Service Worker automatically make an app work offline?**

No. The app must explicitly cache required resources and define fetch behaviour for cached, network and failure cases. Offline functionality also needs a strategy for fresh data, cache updates and unavailable resources.

### Performance and design

**Q17. When would you use a worker instead of `setTimeout`?**

A timer schedules work on the same main thread; it does not move computation off-thread. Use a worker when CPU-heavy work would block the page. Use timers to defer or schedule smaller main-thread work.

**Q18. What are the costs of using a worker?**

Worker startup, memory, message serialization/copying, coordination, error handling and lifecycle management. For small tasks, these costs can outweigh the benefit.

**Q19. How would you process a very large CSV without freezing the UI?**

Parse it in a Dedicated Worker, process in chunks, and send incremental results or progress messages back. Avoid repeatedly sending the entire parsed structure. Consider transferable buffers and cancellation. Keep DOM rendering and UI updates on the main thread.

**Q20. How would you synchronize login state across tabs?**

Use a suitable cross-tab mechanism such as BroadcastChannel or a Shared Worker to notify tabs of state changes. The server remains the source of truth for authentication; do not rely on client-side messages as authorization.

**Q21. How would you design offline caching for a PWA?**

Precache the app shell during installation, version caches, delete obsolete caches during activation, and choose a strategy per resource type. Cache-first can suit versioned static assets; network-first may suit frequently changing data. Provide an offline fallback and test update and stale-data behaviour.

**Q22. Can you use React hooks inside a worker?**

No. Hooks must run in React's component/rendering lifecycle. A worker is a separate JavaScript context. The page can communicate through messages and update React state from message handlers.

**Q23. Can a worker create another worker?**

Workers can create workers in environments that support nested workers, subject to URL and browser constraints. Consider whether additional complexity and resource use are justified.

**Q24. What is the difference between a Web Worker and a Service Worker?**

Web Worker is the general family of background execution APIs, including Dedicated and Shared Workers. Service Worker is a specialized worker with a registration-based, event-driven lifecycle and network-interception capabilities.

## 6. Real-world scenario mapping

| Scenario | Good starting point | Why |
|---|---|---|
| Parse a multi-megabyte CSV | Dedicated Worker | Keep parsing off the UI thread |
| Perform image transformations | Dedicated Worker | CPU-heavy computation |
| Coordinate multiple same-origin tabs | Shared Worker or BroadcastChannel | Cross-context messaging |
| Cache app shell for offline use | Service Worker | Cache Storage and fetch handling |
| Intercept navigation requests | Service Worker | Scoped fetch event |
| Push notification handling | Service Worker | Push event support |
| Keep React components responsive | Dedicated Worker for heavy computation | UI updates remain on main thread |

## 7. Common mistakes and follow-up points

- **Mistake:** Saying a worker makes every task faster. **Correction:** It improves responsiveness when work is expensive enough to justify the overhead.
- **Mistake:** Saying a Service Worker is always running. **Correction:** It is browser-managed and event-driven; it may be stopped while idle.
- **Mistake:** Saying Shared Workers synchronize tabs automatically. **Correction:** They provide a shared communication endpoint; the application implements synchronization.
- **Mistake:** Saying Service Workers cache all API responses automatically. **Correction:** The application explicitly implements caching and fetch strategies.
- **Mistake:** Saying workers can use `window` or manipulate the DOM. **Correction:** They have no page DOM access.
- **Mistake:** Treating `postMessage()` as shared object references. **Correction:** Messages are normally structured-cloned unless ownership is transferred or shared memory is used.

## 8. Last-minute revision sheet

| Remember | One-line answer |
|---|---|
| Dedicated | One owner; background computation |
| Shared | Multiple same-origin contexts; MessagePorts |
| Service | Registration/scope; event-driven network proxy |
| Communication | `postMessage()` |
| Worker global | `self` |
| DOM access | Not available |
| Large binary payload | Transferable `ArrayBuffer` |
| Shared memory | `SharedArrayBuffer` + `Atomics`, with isolation requirements |
| Offline support | Service Worker + Cache Storage |
| Secure context | HTTPS, with localhost exception |
| Stop Dedicated Worker | `terminate()` or worker calls `close()` |
| Storage | IndexedDB available; `localStorage` unavailable in workers |

### Explain in 30 seconds

"Web Workers let us move JavaScript work away from the browser's main thread. I use a Dedicated Worker for heavy computation owned by one page, a Shared Worker when multiple same-origin tabs need to communicate through one worker, and a Service Worker for event-driven capabilities like fetch interception, offline caching and push. Workers cannot manipulate the DOM; the page and worker communicate through messages."

## 9. Self-check before marking complete

- [ ] Explain why a long synchronous task freezes the UI.
- [ ] Explain the differences among all three worker types without notes.
- [ ] Write a Dedicated Worker message round-trip.
- [ ] Explain Shared Worker MessagePorts.
- [ ] Explain Service Worker install, activate and fetch.
- [ ] Explain structured clone vs transferable objects.
- [ ] Explain why Service Workers need HTTPS.
- [ ] Give one real-world use case for each worker.
- [ ] Answer at least 10 questions aloud.
- [ ] Explain trade-offs and failure cases.

## 10. Further reading

- [MDN: Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- [MDN: Worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker)
- [MDN: SharedWorker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [HTML Standard: Web workers](https://html.spec.whatwg.org/multipage/workers.html)
- [web.dev: Service worker lifecycle](https://web.dev/articles/service-worker-lifecycle)
