# JavaScript Event Loop — Interview Preparation

**Priority:** P0 · **Level:** SDE-1 to Senior · **Category:** JavaScript / Browser internals

> **Revision slogan:** Run synchronous code first, drain microtasks, then the browser gets opportunities to render and process the next task.

## 1. Why does the event loop exist?

JavaScript code in a browser page usually runs on the main thread. The browser must also respond to user input, update the DOM and render frames. If JavaScript blocks the main thread with a long synchronous task, the page becomes unresponsive.

The event loop coordinates execution between the JavaScript call stack, tasks (often called macrotasks in interviews), the microtask queue, and browser rendering.

JavaScript is often described as single-threaded because a given agent has one main JavaScript execution thread. The browser itself is multi-process and multi-threaded, and workers run in separate agents.

## 2. Core components

### Call stack
The call stack tracks currently executing JavaScript function calls. JavaScript runs synchronously until the current stack becomes empty or execution yields.

### Tasks (macrotasks)
A task is a unit of work scheduled by the host environment. Common examples include timer callbacks from setTimeout/setInterval, user interaction callbacks, message events, and some script execution and browser event callbacks.

The term **macrotask** is common interview shorthand; the HTML Standard generally describes task queues and task sources.

### Microtasks
Microtasks run after the current JavaScript stack completes and before the event loop proceeds to the next task. Common sources:
- Promise.then(), catch() and finally() callbacks.
- queueMicrotask().
- MutationObserver callbacks.

The browser drains the microtask queue, including microtasks queued by other microtasks, before moving on. An endlessly replenished microtask queue can starve rendering and other tasks.

### Rendering opportunity
After a task and its microtask checkpoint, the browser may update rendering when appropriate. Rendering is not guaranteed after every task; the browser controls frame scheduling.

## 3. The execution model

A useful simplified model:

1. Execute one task.
2. Run JavaScript until the call stack is empty.
3. Perform a microtask checkpoint: drain microtasks until empty.
4. The browser may update rendering.
5. Select and run another task, then repeat.

This is a mental model, not a promise that every browser frame follows an identical fixed sequence. The HTML event loop and rendering steps have more detail.

## 4. Example: predict the output

~~~js
console.log("A");

setTimeout(() => console.log("B"), 0);

Promise.resolve().then(() => console.log("C"));

queueMicrotask(() => console.log("D"));

console.log("E");
~~~

**Output:**

~~~text
A
E
C
D
B
~~~

**Why?**
- A and E are synchronous, so they run first.
- The Promise callback and queueMicrotask callback are microtasks. They run in enqueue order.
- The timer callback is a later task, so it runs after the microtask checkpoint.

A zero-millisecond timer does not mean “run immediately”; it means the callback becomes eligible after the timer's delay and scheduling rules.

## 5. Promise chains and microtask ordering

~~~js
console.log("start");

Promise.resolve()
  .then(() => {
    console.log("promise 1");
    queueMicrotask(() => console.log("nested microtask"));
  })
  .then(() => console.log("promise 2"));

queueMicrotask(() => console.log("queued microtask"));

console.log("end");
~~~

**Output:**

~~~text
start
end
promise 1
queued microtask
nested microtask
promise 2
~~~

Reasoning:
1. The first Promise reaction is queued.
2. The explicit microtask is queued after it.
3. The first reaction runs and queues the nested microtask. Its returned promise settles, queuing the next then reaction.
4. The already queued explicit microtask runs.
5. The nested microtask runs.
6. The second Promise reaction runs.

## 6. async / await and the event loop

An async function always returns a Promise. When execution reaches await, the function pauses and its continuation is scheduled as a Promise reaction (microtask) when the awaited value settles.

~~~js
console.log("1");

async function run() {
  console.log("2");
  await Promise.resolve();
  console.log("4");
}

run();

console.log("3");
~~~

**Output:**

~~~text
1
2
3
4
~~~

The code before await runs synchronously. The continuation after await runs asynchronously through the Promise job/microtask mechanism.

## 7. Browser vs Node.js event loop

Do not blindly apply browser ordering rules to Node.js.

| Concept | Browser | Node.js |
|---|---|---|
| Main scheduling model | HTML event loop | libuv event loop |
| Promise callbacks | Microtasks | Microtasks |
| queueMicrotask | Microtasks | Microtasks |
| Timer callbacks | Timer tasks | Timers phase |
| setImmediate | Not a standard browser API | Node-specific check phase |
| process.nextTick | Not available | Node-specific queue with special priority |
| Rendering | Browser rendering opportunities | No browser page rendering loop by default |

Node's ordering can also depend on context and Node version. In interviews, clarify whether the question is about a browser or Node.js runtime.

## 8. Common interview questions and answers

**Q1. What is the event loop?**

The event loop is the host runtime mechanism that coordinates synchronous JavaScript execution with queued asynchronous callbacks. In browsers it works with tasks, microtasks and rendering opportunities so the page can handle events and remain responsive.

**Q2. What is the difference between a task and a microtask?**

A task is a unit of scheduled work such as a timer or event callback. A microtask is higher-priority follow-up work, commonly from Promise reactions or queueMicrotask. After a task's JavaScript completes, the browser drains microtasks before proceeding to another task and a possible rendering opportunity.

**Q3. Why does Promise.then run before setTimeout(..., 0)?**

Promise reactions are microtasks. Timer callbacks are scheduled as tasks. The microtask checkpoint is drained before the event loop moves on to a later task. The timer delay is also a minimum threshold, not an exact execution time.

**Q4. Does setTimeout(fn, 0) execute immediately?**

No. It schedules a timer callback that becomes eligible after the minimum delay and browser scheduling constraints. It cannot interrupt currently running JavaScript.

**Q5. What happens when a microtask queues another microtask?**

The newly queued microtask is processed in the same microtask checkpoint before the next task. A chain that keeps adding microtasks can starve rendering and user input.

**Q6. Can the browser render between two synchronous statements?**

No. A long-running JavaScript task prevents the main thread from reaching a rendering opportunity. The browser generally needs the main thread to become available to perform relevant rendering work.

**Q7. Does the browser render after every task?**

No. Rendering updates are browser-controlled and may be skipped or coalesced. The event loop provides rendering opportunities, not a guarantee of a paint after every task.

**Q8. What does await do to execution?**

It suspends the async function's continuation until the awaited value settles. The code before await runs synchronously; the continuation runs asynchronously as a Promise reaction.

**Q9. What is queueMicrotask used for?**

It schedules a callback in the microtask queue without creating a Promise. It is useful when code needs to run after the current synchronous work but before the next task.

**Q10. What is a long task?**

In browser performance tooling, a Long Task is a task that occupies the main thread for more than 50 ms. It can delay input response and rendering. Break work into chunks or move CPU-heavy work to a Web Worker.

**Q11. How can you yield to the browser between chunks of work?**

Split work into smaller units and schedule subsequent chunks with a task-yielding mechanism such as setTimeout (or scheduler.yield() where supported). A Promise microtask alone does not yield to rendering because the microtask queue is drained before the next task.

**Q12. Does JavaScript run on one thread in the browser?**

A page's main JavaScript agent typically has one main execution thread, but browsers use multiple threads and processes. Web Workers run JavaScript in separate agents and can execute in parallel.

**Q13. What is the difference between async and non-blocking?**

An async API lets the caller continue without waiting synchronously for the operation to complete. Non-blocking means the operation does not hold up the current thread while waiting. CPU-heavy synchronous work remains blocking even if placed inside an async function.

**Q14. Does making a function async move it to another thread?**

No. An async function executes synchronously on the current thread until it suspends at an await. Use a Web Worker for CPU-intensive work that must run off the main thread.

**Q15. Why can recursive Promise.then calls freeze a page?**

Each Promise reaction queues another microtask. Since the browser drains microtasks until the queue is empty, an infinite microtask chain can prevent the browser from reaching rendering and the next task.

**Q16. What is the difference between Promise callbacks and MutationObserver callbacks?**

Both are delivered through the microtask mechanism. Promise callbacks are reactions to Promise settlement; MutationObserver callbacks are delivered when observed DOM mutations are processed at a microtask checkpoint.

**Q17. How does requestAnimationFrame fit into the event loop?**

requestAnimationFrame callbacks are scheduled for a browser rendering update, generally before the next paint. They are useful for visual updates and animation, not for arbitrary background computation.

**Q18. How do you debug event-loop ordering?**

Write down synchronous statements, then identify which callbacks are tasks, microtasks or rendering callbacks. Track the enqueue order of microtasks and execute the queue after the current stack empties. Verify assumptions in the target runtime.

## 9. Output-based practice questions

Try to predict each output before expanding the answer.

### Practice 1

~~~js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
~~~

<details>
<summary>Show answer</summary>

~~~text
A
D
C
B
~~~

</details>

### Practice 2

~~~js
console.log("start");

setTimeout(() => console.log("timer"), 0);

Promise.resolve().then(() => {
  console.log("promise");
  setTimeout(() => console.log("timer from promise"), 0);
});

console.log("end");
~~~

<details>
<summary>Show answer</summary>

~~~text
start
end
promise
timer
timer from promise
~~~

The first timer is queued before the Promise callback runs. The Promise callback queues the second timer later.

</details>

### Practice 3

~~~js
console.log("one");

async function test() {
  console.log("two");
  await null;
  console.log("three");
}

test();

Promise.resolve().then(() => console.log("four"));
console.log("five");
~~~

<details>
<summary>Show answer</summary>

~~~text
one
two
five
three
four
~~~

The await continuation is queued before the explicit Promise callback.

</details>

### Practice 4

~~~js
console.log("A");

queueMicrotask(() => {
  console.log("B");
  queueMicrotask(() => console.log("C"));
});

Promise.resolve().then(() => console.log("D"));

console.log("E");
~~~

<details>
<summary>Show answer</summary>

~~~text
A
E
B
D
C
~~~

The microtask B queues C behind D, which was already waiting.

</details>

## 10. Real-world frontend scenarios

| Problem | Event-loop understanding |
|---|---|
| UI freezes while parsing JSON | Parsing is synchronous CPU work; move it to a worker or chunk it |
| Search input feels delayed | Debounce requests and avoid expensive work in input handlers |
| Animation stutters | Avoid long main-thread tasks; use requestAnimationFrame for visual updates |
| Loading spinner does not appear | Synchronous work may prevent the browser from painting it |
| Promise chain delays interaction | Excessive microtasks can starve tasks and rendering |
| Need to process a large list | Chunk work and yield between chunks, or use a worker |

## 11. Last-minute revision sheet

| Concept | Remember |
|---|---|
| Call stack | Executes synchronous JavaScript |
| Task | Timer, user event and other host-scheduled work |
| Microtask | Promise reaction, queueMicrotask, MutationObserver |
| Ordering | Current stack → microtask checkpoint → possible rendering → next task |
| setTimeout 0 | Not immediate; schedules a later task |
| Promise.then | Microtask |
| async/await | Continuation after await is Promise-based microtask work |
| requestAnimationFrame | Callback associated with a rendering update |
| Long task | More than 50 ms of main-thread blocking |
| Worker | Separate agent for off-main-thread JavaScript |

### Explain in 30 seconds

"The event loop coordinates synchronous JavaScript with asynchronous work. The current call stack runs to completion. Then the browser drains microtasks such as Promise callbacks and queueMicrotask. After that, the browser may render before taking another task such as a timer or user event. Long synchronous tasks and endless microtask chains can both hurt responsiveness."

## 12. Self-check
- [ ] Explain call stack, tasks and microtasks.
- [ ] Predict output for Promise and timer combinations.
- [ ] Explain why zero-delay timers are not immediate.
- [ ] Explain how await schedules continuation.
- [ ] Explain microtask starvation.
- [ ] Explain requestAnimationFrame.
- [ ] Compare browser and Node.js scheduling.
- [ ] Explain how to avoid long tasks.

## 13. Further reading
- [MDN: Event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
- [HTML Standard: Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)
- [MDN: queueMicrotask](https://developer.mozilla.org/en-US/docs/Web/API/Window/queueMicrotask)
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)
- [web.dev: Optimize long tasks](https://web.dev/articles/optimize-long-tasks)

---
**Next recommended chapter:** [Browser Rendering Pipeline](../../browser/browser-rendering-pipeline/README.md)
