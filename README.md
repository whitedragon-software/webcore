# Webcore

A single-file Cloudflare Worker that serves its own chat app and runs it on
Workers AI: conversations saved in D1, five free models, optional web search,
optional step-by-step thinking, and regenerated answers that stay reachable as
switchable versions.

```
https://webcore.workers.dev/
```

- `worker.js`: the Worker, chat UI included. Deploy this file as-is; no build
  step, no dependencies.

---

## Deploy

Paste the file into the Worker editor in the Cloudflare dashboard and deploy.
The free tier is enough. No Wrangler is needed.

The Worker needs two bindings (Settings, Bindings in the dashboard) or it
cannot start:

| Variable name | Type | Purpose |
|---|---|---|
| `DB` | D1 database | Conversations, messages and the daily neuron count. Create an empty D1 database first; the Worker creates and migrates its own tables on the first request. |
| `AI` | Workers AI | The models. |

The variable names must match exactly. With a binding missing you get
`Database unavailable` (503) or an AI error instead of a chat.

To update, paste the new file over the old one and deploy. The page is cached
for up to 5 minutes, so a hard refresh may be needed before the new UI shows.

### Optional: web search

The globe button only works once a search key exists. Add one as a secret
(Settings, Variables and Secrets):

| Name | Purpose |
|---|---|
| `TAVILY_API_KEY` | Used first if set. Free tier at tavily.com, no card. |
| `BRAVE_API_KEY` | Used only if Tavily is not set. May require a card. |

**Unset, the app behaves exactly as before:** turning search on shows an error
toast explaining how to add a key, and the question is answered without
search.

### Optional: model license

Some models need their terms accepted once. If you see *Model license not
accepted*, open the dashboard, AI, Models, open that model, and agree.

---

## Using it

Open the Worker's URL. You land on the welcome page ("What can I help
with?"); opening the site never opens an old chat and never creates one. Pick
a conversation in the sidebar to continue it, or just type: the first message
automatically creates a conversation called "New Chat". Press Enter to send
(Shift+Enter for a new line).

| Control | What it does |
|---|---|
| Model dropdown | Llama 4, GPT-OSS, Gemma 4, GLM 4.7, Qwen 3.8. Applies to the next message or regenerate. |
| Globe | Runs a live web search first and answers from the results. Sources are appended to the reply. |
| Lightbulb | Asks the model to reason first. Shows as a collapsible "Thought process" above the answer. |
| Circular arrow on a reply | Regenerate. Adds a new version; the old one is kept. |
| `<` `>` and counter | Switch between versions of a reply. |
| Sidebar | Create, rename, delete and switch conversations. |

**Versions:** sending a message after switching to an older version continues
from that version, the same as claude.ai.

**Neuron bar:** the bar at the bottom of the sidebar shows today's usage
against the daily cap (`DAILY_NEURON_LIMIT`, default 10,000, UTC day). Requests
that would exceed it are refused with the remaining amount.

**HTML replies:** a reply containing an HTML file gets a block with Code and
Preview tabs, Copy and Download. The preview runs in a sandboxed iframe.

---

## How it works

**Branching.** Every message row has a `parent_id`, and each parent stores
which child is active (`active_child_id`). Regenerating inserts a new sibling
under the same parent and never overwrites anything, so every version stays
reachable after a reload or from another device.

**Long-conversation memory.** The last 30 messages go to the model
word-for-word. Older ones are folded into a running summary stored on the
conversation, refreshed after 20 more older messages pile up. Anything the
summary does not yet cover is sent verbatim, so no message falls into a gap.

**Server side.**
- Tables are created and migrated once per isolate; a failed init is retried
  by the next request. Migrations only run when a column is actually missing,
  and indexes are created after them so older databases do not break.
- All system messages are merged into one, because several models reject more
  than one.
- Reasoning tags are removed from earlier replies before they are sent back to
  the model. Otherwise a reply made with thinking on teaches the model to keep
  "thinking" later, including when you regenerate with thinking off.
- One retry for transient AI errors. License, 4xx and quota errors are not
  retried. Empty model replies become a clear error instead of a blank message.
- Work is registered with `ctx.waitUntil`, so a reply is still saved if the
  browser disconnects.
- If an interrupted write leaves a message without an active child, the newest
  child is followed instead of cutting the conversation off.
- The neuron count comes from the model's usage report when it has one,
  otherwise from an estimate.
- Search results are used for that one call only and never stored. Each
  snippet is capped at 600 characters and the model is told to treat results
  as untrusted.
- `/api/chat` and `/api/regenerate` stream newline-delimited JSON: status
  events ("Searching the web...", "Thinking..."), then one result. The reply
  itself is not token-streamed; it arrives whole.

**Client side (embedded UI).** A small markdown renderer (headings, lists,
tables, links, code blocks), the version navigator, the model, search and
thinking toggles, theme saved in `localStorage`, and swipe gestures for the
sidebar on phones. A failed send restores what you typed and reloads the
conversation, so no phantom message is left behind. Code blocks cut off by the
token limit are still shown.

---

## Editing and verifying `worker.js`

The UI is a long string of concatenated single-quoted pieces, and its
JavaScript lives inside that string, so **`node --check worker.js` does not
validate it.** A syntax error there breaks the whole page in the browser while
the outer file still checks clean. Always run both:

```
cp worker.js worker.mjs
node --check worker.mjs

node -e "
import('./worker.mjs').then(async m => {
  const r = await m.default.fetch(new Request('https://x/'), {}, { waitUntil() {} });
  const html = await r.text();
  const s = html.match(/<script>([\s\S]*)<\/script>/)[1];
  new Function(s.replace(/document\.getElementById\([^)]*\)/g, '({})'));
  console.log('UI script parses');
});
"
```

Inside the UI string, use single quotes only for the string pieces (escape any
`'` in UI code), and double every backslash (`\\n`, `\\u0000`). In UI code, do
not pass code text as a string to `.replace()`; use a function
(`.replace(x, function () { return y })`), because `$&` and similar patterns in
a replacement string corrupt code.

To add a model, change all three places together: `FREE_MODELS`, the `<select>`
options, and the `MODEL_NAMES` map in the UI. If they disagree, the dropdown
offers a model the server rejects.

---

## Quick test checklist

1. Homepage opens on the welcome page ("What can I help with?"), not on an old
   chat, and the sidebar neuron bar fills in.
2. Send a message: a conversation called "New Chat" is created and the reply
   appears.
3. Reload: the conversation and reply are still there.
4. Regenerate: a `2/2` counter appears and `<` returns to the original.
5. Switch to version 1 and send another message: it continues from version 1.
6. Ask for a long HTML page: a Code / Preview block appears even if the reply
   hit the token limit.
7. Lightbulb on: a "Thought process" block appears above the answer.
8. Regenerate that reply with the lightbulb off: the old "Thought process"
   block disappears at once and the new reply has none.
9. Globe on with a key set: a Sources list is appended. Without a key: an error
   toast explains how to add one.
10. Remove the `AI` or `DB` binding temporarily: you get a clear error, not a
   blank page.

---

## Limitations

- **No login.** Anyone with the URL can read, rename and delete every
  conversation and use up the daily neurons, and `CORS_ORIGIN` is `*`. Put
  Cloudflare Access in front of the Worker, or at least set `CORS_ORIGIN` to
  your own domain, before sharing the URL.
- **One shared history.** There are no accounts; every visitor sees the same
  conversation list.
- **Neuron counts are estimates.** The pre-check counts only the new prompt and
  search text, not the history, and summary calls are not added to the daily
  total.
- **Summaries are per conversation, not per branch.** A summary made on one
  branch can be reused on another of similar length. One that covers more
  messages than the new path has is discarded.
- **Pre-branching data.** Messages saved before branching support all have no
  parent, so they show as several roots and only the first is displayed.
- **Length caps.** 2,000 characters in, and 800 to 1,000 tokens out depending
  on the model. Long answers are cut off.
- **Search uses your raw message text**, not a rewritten query, so follow-ups
  like "what about the second one?" search poorly.
- **Thinking mode is prompt-based.** Weaker models sometimes ignore the
  `<thinking>` format and the reasoning then shows as part of the answer.
- `MAX_HISTORY` in `CONFIG` is not used by anything.
- Unconfirmed on a live deployment (checked for syntax and tested in isolation
  only, not against real D1 and Workers AI): merged system messages, the AI
  retry, the memory gap fix, DB init and migration, `waitUntil`, and the three
  UI fixes. Use the checklist above after deploying.
