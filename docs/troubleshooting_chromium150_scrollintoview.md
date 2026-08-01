# "Unexpected Application Error!" on Chromium 150+ (`TypeError: <x> is not a function`)

## Symptom

The deployed app loads normally, but as soon as the user asks a question the entire page is
replaced by React Router's error screen:

```
Unexpected Application Error!
TypeError: Ae is not a function
    at yf (/assets/vendor-<hash>.js:48:93926)
    at YA (/assets/vendor-<hash>.js:48:111329)
    at ym ...
```

Distinguishing characteristics that make this bug look like an infrastructure problem when it is not:

- The app worked for months and started failing without any deployment.
- It reproduces on a brand-new machine, in InPrivate mode, with no extensions.
- Clearing the cache does not help.
- It does not reproduce in older browsers or in headless Chromium.
- The initial page load is fine — it only breaks *after* the first question.

## Root cause

**Chromium 150 changed `Element.scrollIntoView()` to return a `Promise`** (scroll-completion
promises). It previously returned `undefined`.

[Chat.tsx](../app/frontend/src/pages/chat/Chat.tsx) called it from expression-bodied
`useEffect` arrow functions:

```tsx
// BROKEN on Chromium >= 150
useEffect(() => chatMessageStreamEnd.current?.scrollIntoView({ behavior: "smooth" }), [isLoading]);
useEffect(() => chatMessageStreamEnd.current?.scrollIntoView({ behavior: "auto" }), [streamedAnswers]);
```

An expression-bodied arrow function **returns** the value of the expression. React treats any
non-`undefined` return value from an effect as that effect's **cleanup function** and invokes it
during teardown. `react-dom` only guards `void 0 !== destroy` before calling it, so on unmount it
tries to call a `Promise` as a function:

```
TypeError: Ae is not a function
```

React Router's root `errorElement` catches the throw and replaces the whole page.

Why only after asking a question: before the first message, `chatMessageStreamEnd.current` is
`null`, so `?.` short-circuits and the effect returns `undefined`. Once the ref is populated,
every run of the effect registers a `Promise` as the cleanup.

## Fix

Use block bodies so nothing is returned to React — see
[Chat.tsx](../app/frontend/src/pages/chat/Chat.tsx):

```tsx
// Use block bodies so nothing is returned to React. An expression-bodied effect returns
// whatever scrollIntoView() evaluates to, and React treats any non-undefined return value
// as the cleanup function and invokes it on unmount ("TypeError: <x> is not a function").
useEffect(() => {
    chatMessageStreamEnd.current?.scrollIntoView({ behavior: "smooth" });
}, [isLoading]);
useEffect(() => {
    chatMessageStreamEnd.current?.scrollIntoView({ behavior: "auto" });
}, [streamedAnswers]);
```

A secondary fix was applied to the `/` route in [app.py](../app/backend/app.py). `index.html`
references content-hashed asset filenames that change on every deploy, so it must be revalidated
on each load, otherwise a cached `index.html` can point at chunks from a previous deployment:

```python
@bp.route("/")
async def index():
    response = await bp.send_static_file("index.html")
    response.headers["Cache-Control"] = "no-cache, must-revalidate"
    return response
```

## Guideline

**Never use an expression-bodied arrow function as a `useEffect` callback** unless the expression
genuinely evaluates to a cleanup function. Any DOM API can start returning a value in a future
browser version, and React will silently treat it as a cleanup function.

```tsx
useEffect(() => doSomething(), [dep]);      // risky
useEffect(() => { doSomething(); }, [dep]); // safe
```

Browser APIs that have gained or may gain return values and are commonly called this way:
`scrollIntoView()`, `scrollTo()`, `scroll()`, `scrollBy()`, `requestFullscreen()`,
`element.animate()`.

## How it was diagnosed

Standard debugging did not help because the minified frames pointed into `react-dom` internals.
The technique that worked:

1. Drive the **exact browser build that reproduces the bug** with Playwright
   (`p.chromium.launch(channel="msedge", headless=False)`). Headless Chromium 150 does **not**
   reproduce it — it must be headed.
2. Before page load, inject a shim for `window.__REACT_DEVTOOLS_GLOBAL_HOOK__` that implements
   `inject` / `onCommitFiberRoot`, then walk the fiber tree on every commit looking for effect
   hooks whose `memoizedState.inst.destroy` is neither `undefined` nor a function.

That surfaced the offending hook directly:

```json
{ "component": "Wu", "hookIndex": 60, "effectTag": 8,
  "value": { "type": "object", "ctor": "Promise" } }
```

A direct API probe confirmed the browser behaviour change:

```js
typeof document.body.scrollIntoView({ behavior: "smooth" });
// Chromium 149 and earlier: "undefined"
// Chromium 150:             Promise
```

## Verification

After rebuilding the frontend (`npm run build` in `app/frontend`) and deploying
(`azd deploy backend`), the same headed-Edge-150 script reported no crash and no bad cleanup
values across every unmount path: initial load, after asking a question, after clearing the chat,
after a route change, and after navigating back.
