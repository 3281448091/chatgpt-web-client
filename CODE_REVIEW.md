# Code Review — chatgpt-web-client

Scope: the whole codebase at `ba28bf4` (`app.vue`, `components/Chat.vue`, `nuxt.config.js`, `tailwind.config.js`, `package.json`). Nearly all of the logic is in `components/Chat.vue`, so that is where the findings are.

Findings are ordered by severity. "Confirmed" means I traced the failure through the code (including the `@microsoft/fetch-event-source@2.0.1` source); "Plausible" means it depends on what the API server returns.

---

## 1. Switching browser tabs mid-reply sends the message again — High, Confirmed

`components/Chat.vue:94`

`fetchEventSource` is called without `openWhenHidden: true`. By default the library listens for `visibilitychange`, and when the tab becomes hidden it **aborts the request**. When the tab is visible again it calls `create()`, which **re-POSTs the same body**.

**Scenario:** the user sends a long question, switches to another tab while the reply streams, then comes back. The server gets the same message a second time and starts a new reply. Because `botMessage.text += …` keeps appending, the second reply is glued onto the cut-off first one. With node-chatgpt-api this also adds a duplicate turn to the server-side conversation and costs a second API call.

**Fix:** pass `openWhenHidden: true` in the `fetchEventSource` options.

## 2. Clicking "Stop" throws a TypeError in `finally` — Medium, Confirmed

`components/Chat.vue:39-45`, `components/Chat.vue:160-166`

`stopProcessing()` aborts the controller and sets `processingController.value = null`. The abort makes `fetchEventSource` resolve, so the `finally` block in `sendMessage` runs and reads `processingController.value.signal.aborted` while `processingController.value` is `null`. That throws `TypeError: Cannot read properties of null (reading 'signal')`, which ends up as an unhandled promise rejection, and `inputTextElement.value.focus()` never runs. Unmounting the component during a request hits the same path through `onUnmounted → stopProcessing`.

**Fix:** keep a local reference, e.g. `const controller = new AbortController(); processingController.value = controller;`, and use `controller` in `onmessage` and `finally`. Alternatively, guard with `processingController.value?.signal.aborted`.

## 3. Failed requests are never shown to the user; the bubble shows a blinking cursor forever — Medium, Confirmed

`components/Chat.vue:97-110`, `components/Chat.vue:158-159`, `components/Chat.vue:257`

A non-200 response (`onopen`), a connection the server closes before `[DONE]` (`onclose`), a network error, or an exception inside `onmessage` all end up in `catch (err) { console.log('ERROR', err); }`. None of them update `botMessage`. The template renders a message as "streaming" (with the `█` cursor) whenever `message.raw` is unset, so the reply stays as an empty bubble with a cursor and no error text. A stopped reply (finding 2) looks the same.

**Fix:** in `catch`, set `botMessage.error = true`, `botMessage.text ||= err.message`, and mark the message as finished. A `done`/`streaming` flag is clearer than checking `raw`, and it also makes "Stop" render properly.

## 4. Pressing Enter to confirm IME input (Chinese/Japanese/Korean) sends the message — Medium, Confirmed

`components/Chat.vue:292`

`@keydown.enter.exact.prevent="sendMessage(message)"` fires while an IME composition is still active. With Pinyin, for example, pressing Enter to commit the candidate sends the half-typed text instead.

**Fix:** ignore Enter while composing:

```vue
@keydown.enter.exact="e => { if (!e.isComposing) { e.preventDefault(); sendMessage(message); } }"
```

(Safari also needs a `keyCode === 229` check.)

## 5. `crypto.randomUUID()` breaks sending when the app is served over plain HTTP — Medium, Confirmed

`components/Chat.vue:62`, `components/Chat.vue:71`

`crypto.randomUUID` only exists in secure contexts (HTTPS or `localhost`). A common self-hosted setup is opening the client at `http://192.168.x.x:3000` from another device. There, `sendMessage` throws on line 62 before anything is shown, so clicking Send appears to do nothing.

**Fix:** these IDs are only used as Vue `:key`s, so an incrementing counter (`let nextId = 0; … id: nextId++`) is enough.

## 6. `result.details` is assumed to exist — Low/Medium, Plausible

`components/Chat.vue:132`, `components/Chat.vue:140`

`result.details.adaptiveCards?.…` and `result.details.suggestedResponses` throw if a backend (or a compatible server, which the README invites) leaves out `details`. The throw happens after `conversationData` has been updated but before `botMessage.raw` is set, so the reply is left with the cursor forever (see finding 3).

**Fix:** use `result.details?.adaptiveCards?.…` and `result.details?.suggestedResponses`.

## 7. Every streamed token re-renders and re-highlights the whole conversation — Low (performance), Confirmed

`components/Chat.vue:257`

`parseMarkdown(...)` is called inline in `v-html` for every message. Each token changes `botMessage.text`, which re-renders the list and runs `marked.parse` + `hljs.highlightAuto` + `DOMPurify.sanitize` on **every** message in the history. `highlightAuto` is especially expensive because it tries every registered language. In long, code-heavy conversations, streaming gets noticeably slower.

**Fix:** cache the rendered HTML on finished messages (for example, compute `message.html` once when the message is done) and only re-parse the message that is currently streaming. Or move the rendering into a small child component with a `computed`.

## 8. The code-highlight callback ignores the declared language — Low, Confirmed

`components/Chat.vue:12-19`

The `try` branch always calls `hljs.highlightAuto(code)`, and that almost never throws, so the `lang` from the code fence (```` ```python ````) is effectively ignored. Auto-detection often guesses wrong on short snippets.

**Fix:** reverse the order. Use `hljs.highlight(code, { language: lang })` when `hljs.getLanguage(lang)` is true, and fall back to `highlightAuto` otherwise.

---

## Nits / cleanup

- `nuxt.config.js:2` — `tailwindTypography` is imported but never used (the plugin is registered in `tailwind.config.js`).
- `nuxt.config.js:5` / `package.json` — the `nuxt-icon` module is installed and registered, but no `<Icon>` is used anywhere.
- `components/Chat.vue:217` — the `if (!process.server)` guard is unnecessary. `onMounted`/`onUnmounted` never run on the server.
- `components/Chat.vue:98,105,131,134,159` — `console.log`/`console.debug` calls are left in, and they log full API responses.
- `nuxt.config.js:10` — if `API_BASE_URL` is unset, requests go to `undefined/conversation`. A default (`'http://localhost:3000'`, as in `.env.example`) or a startup warning would help. The README could also mention that `NUXT_PUBLIC_API_BASE_URL` overrides it at runtime for production builds.
- `components/Chat.vue:281` — `:key="response"` causes duplicate-key warnings if the server ever sends two identical suggestions. Using the index is safer here.

## Things that look good

- Markdown output is always passed through `DOMPurify.sanitize` before `v-html`, including user messages. The citation regex rewrites happen *before* sanitization, so they can't introduce unsafe HTML.
- Sending is correctly blocked while a request is already in flight (`components/Chat.vue:48`).
- `onerror` rethrows, which correctly turns off the library's automatic retry loop. That is the right call for a non-idempotent POST.

## Suggested priority

Fix 1–3 first. They are small, local changes in `sendMessage` and affect every user. 4 and 5 matter a lot for specific audiences (CJK input, LAN self-hosting) and are one-liners. 6–8 and the nits can be batched into a cleanup.
