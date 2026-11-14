# Frontend Interview Preparation

A practical, topic-wise guide for SDE-1, SDE-2 and Senior Frontend Engineer interviews. Each chapter includes core concepts, runnable examples, common pitfalls, interview questions with answers, real-world scenarios and revision notes.

## How to use
1. Follow the priority order, adapting it to your target role.
2. Explain each concept in your own words.
3. Run and modify the examples.
4. Practise questions aloud and justify trade-offs.
5. Mark a topic complete only when you can explain and implement it without notes.

## Topic roadmap

| Area | Planned coverage | Priority |
|---|---|---|
| JavaScript | Fundamentals, scope, closures, this, prototypes, arrays, coercion | P0 |
| Async JavaScript | Event loop, promises, async/await, timers, cancellation | P0 |
| Browser internals | Rendering, events, storage, workers, networking | P0–P1 |
| HTML & CSS | Semantics, accessibility, forms, layout, responsive design | P0 |
| React | Rendering, reconciliation, hooks, state, Suspense, error boundaries | P0 |
| TypeScript | Types, narrowing, generics, utility types, inference | P0–P1 |
| Performance | Core Web Vitals, profiling, code splitting, caching, virtualization | P1 |
| Frontend architecture | Component architecture, state boundaries, microfrontends, monorepos | P1–P2 |
| Networking & security | HTTP, CORS, cookies, auth, XSS, CSRF, CSP, WebSockets, SSE | P1 |
| Testing | Unit, integration, component and end-to-end testing | P1 |
| Machine coding | Autocomplete, data table, file explorer, infinite scroll, chat | P0–P1 |
| Frontend system design | Requirements, APIs, data flow, scaling, reliability, trade-offs | P1–P2 |

## Prioritized checklist

### P0 — Must know
- [ ] JavaScript execution model, scope, closures, hoisting and `this`
- [ ] Event loop, microtasks, macrotasks and promises
- [ ] Array/object methods and common polyfills
- [ ] Debounce, throttle, currying and memoization
- [ ] HTML semantics, CSS layout and responsive design
- [ ] React fundamentals and hooks
- [ ] TypeScript fundamentals
- [ ] API integration and loading/error/empty states
- [ ] Common machine-coding patterns

### P1 — Strong SDE-II readiness
- [ ] Browser internals and rendering pipeline
- [ ] Web Workers: Dedicated, Shared and Service Workers
- [ ] Browser storage and caching
- [ ] HTTP, CORS, cookies and authentication
- [ ] React performance and rendering behaviour
- [ ] Testing and accessibility
- [ ] Virtualization, pagination and infinite scrolling
- [ ] WebSockets and Server-Sent Events
- [ ] Frontend architecture and system design

### P2 — Senior-level depth
- [ ] Microfrontend and monorepo trade-offs
- [ ] Advanced performance measurement and Core Web Vitals
- [ ] Large-scale state and data synchronization
- [ ] Security threat modelling
- [ ] Resilience, observability and error handling
- [ ] SSR, SSG, CSR, streaming and hydration trade-offs
- [ ] Architecture decisions under scale and organizational constraints

## Repository structure
- `javascript/` — language fundamentals and asynchronous JavaScript
- `browser/` — browser APIs, rendering, storage, workers and networking
- `html-css/` — markup, styling, layout and accessibility
- `react/` — React concepts, hooks, internals and optimization
- `typescript/` — TypeScript concepts and patterns
- `performance/` — profiling and optimization
- `frontend-system-design/` — architecture and system design
- `machine-coding/` — hands-on UI implementation exercises
- `testing/` — testing strategies and tools
- `last-minute-revision/` — compact revision sheets

## Chapters
### JavaScript
- [JavaScript Event Loop](javascript/event-loop/README.md)

### Browser Internals
- [Web Workers](browser/web-workers/README.md)
- [Browser Rendering Pipeline](browser/browser-rendering-pipeline/README.md)
- [Critical Rendering Path](browser/critical-rendering-path/README.md)

### Security
- [Frontend Security: Concepts & Interview Questions](security/README.md)

## Suggested study sequence
JavaScript fundamentals → Event Loop → Browser Rendering Pipeline → Critical Rendering Path → HTML/CSS → React → TypeScript → Performance → Testing → Machine coding → Frontend system design.

Keep explanations interview-focused, examples runnable and caveats explicit. Prefer authoritative specifications and documentation for browser behaviour.
