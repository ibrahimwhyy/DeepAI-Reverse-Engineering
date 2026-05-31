# Build Your Own Free AI Chatbot with the DeepAI API
### A complete A‑to‑Z guide — reverse engineering, backend, frontend, deployment

> **What you'll build:** a fully working web + CLI chatbot that talks to a real
> large language model, streams replies token‑by‑token, remembers the
> conversation, and costs **$0** — no API key, no credit card, no signup.
>
> **How:** by reverse‑engineering the network flow behind `https://deepai.org/chat`
> and reproducing the exact request the website itself sends.
>
> **Time:** ~30–60 minutes. **Level:** intermediate (basic JS/Node).

---

## Table of contents
1. [Legal & ethical notice](#0-legal--ethical-notice)
2. [How it works (the big picture)](#1-how-it-works-the-big-picture)
3. [Part A — Reverse engineering DeepAI](#part-a--reverse-engineering-deepai-step-by-step)
4. [Part B — The core client library](#part-b--the-core-client-library)
5. [Part C — The backend server](#part-c--the-backend-server)
6. [Part D — The frontend UI](#part-d--the-frontend-ui)
7. [Part E — The CLI bot](#part-e--the-cli-bot)
8. [Part F — Run it](#part-f--run-it)
9. [Part G — Deploy it](#part-g--deploy-it-free-hosting)
10. [Part H — Troubleshooting](#part-h--troubleshooting)
11. [API reference](#api-reference)
12. [FAQ](#faq)

---

## 0. Legal & ethical notice

This guide is **educational** — its purpose is teaching how to analyze and
reproduce a web API, the same skill used in QA, security research, and
integration work.

- The code reproduces **exactly the request a normal anonymous browser sends** to
  deepai.org. It does **not** spoof IPs, forge fingerprints, or defeat anti‑bot
  protection.
- DeepAI's free tier is **rate‑limited per IP/cookie**. Respect it. Don't scrape
  at scale, resell access, or hammer the endpoint.
- Undocumented endpoints can change or break at any time. This is **not** a
  stable production API — treat it as a learning project / prototype.
- Read and honor [DeepAI's Terms of Service](https://deepai.org/terms). If you
  need reliability or commercial use, buy their official API.

---

## 1. How it works (the big picture)

A normal hosted AI chatbot looks like this:

```
Browser ──▶ Your Server ──▶ OpenAI/Anthropic (needs a $ API key) ──▶ reply
```

Ours replaces the paid provider with DeepAI's **public free chat endpoint**:

```
Browser ──▶ Your Node server ──▶ api.deepai.org/hacking_is_a_serious_crime ──▶ streamed reply
                 │
                 └── generates the "api-key" itself (client-side algorithm)
```

The trick: DeepAI's anonymous "try it" chat authenticates with an `api-key`
header that is **computed entirely in the browser** by a JavaScript function.
There's no server‑issued secret. Once you understand the algorithm, you can
generate valid keys yourself — for free, forever (subject to rate limits).

```
┌──────────────┐   POST multipart/form-data    ┌────────────────────────┐
│  your client │ ───── api-key: tryit-... ────▶ │ api.deepai.org         │
│  (Node/JS)   │       chatHistory, model       │ /hacking_is_a_serious  │
│              │ ◀──── streamed text reply ──── │ _crime                 │
└──────────────┘                                └────────────────────────┘
```

---

## Part A — Reverse engineering DeepAI (step by step)

This is the part that matters. Here's the exact process used to figure it out —
you can reproduce every step in Chrome DevTools.

### Step 1 — Open the page with DevTools
Go to `https://deepai.org/chat`, press **F12**, open the **Network** tab, tick
**Preserve log**, and send a test message ("hello").

### Step 2 — Find the request
A `POST` appears to a strange URL:

```
POST https://api.deepai.org/hacking_is_a_serious_crime
```

(Yes, that's the real endpoint name — DeepAI's little joke.) The response is a
**streamed plain‑text body**, not JSON.

### Step 3 — Inspect the payload
The request body is `multipart/form-data` with these fields:

| field | value | purpose |
|-------|-------|---------|
| `chat_style` | `"chat"` | UI style |
| `chatHistory` | `[{role,content}, …]` JSON | the whole conversation |
| `model` | `"standard"` / `"online"` / … | which model |
| `session_uuid` | random UUID | conversation id |
| `sensitivity_request_id` | random UUID | moderation id |
| `hacker_is_stinky` | `"very_stinky"` | required literal flag |
| `enabled_tools` | `[]` JSON | image tools (optional) |

### Step 4 — Find the auth header
The only header that matters is:

```
api-key: tryit-82800029597-aae72839f9ef6066914f6a2a41534fcb
```

Where does that come from? Search the page source (Ctrl+F in **Sources**) for
`api-key`. You'll land on calls like `headers:{'api-key':tryitApiKey}` and
`tryitApiKey = generateIslandKey()`.

### Step 5 — Read `generateIslandKey()`
The minified source contains (de‑minified):

```js
function generateIslandKey() {
  let rand = Math.round(Math.random() * 100000000000) + "";
  const H = makeHasher();           // an inline MD5 that returns reversed hex
  return "tryit-" + rand + "-" +
    H(UA + H(UA + H(UA + rand +
      "hackers_become_a_little_stinkier_every_time_they_hack")));
}
```

So the key is:

```
tryit-<rand>-<H( UA + H( UA + H( UA + rand + SALT ) ) )>
```

- `rand` = random 11‑digit number
- `UA`   = `navigator.userAgent`
- `SALT` = `"hackers_become_a_little_stinkier_every_time_they_hack"`
- `H(x)` = **MD5(x) in hex, then the string reversed**

### Step 6 — The key insight
The hash is **MD5** (verifiable: `makeHasher` initializes the classic MD5 magic
constants `1732584193, 4023233417`). The only twist is the digest is
**reversed**. Because everything needed to build the key is in the client, we can
generate valid keys ourselves — that's the whole "auth".

> ⚠️ Because the key is derived from `navigator.userAgent`, your client **must
> send the same `User-Agent` header** it used to compute the key. Mismatch = 401.

### Step 7 — Recap of all tokens
| Token | Source |
|-------|--------|
| `api-key` | computed locally by `generateIslandKey()` |
| `session_uuid` / `sensitivity_request_id` | random UUIDs you create |
| `hacker_is_stinky=very_stinky` | static literal |

That's it. Compare this to ChatGPT (which needs CSRF + a sentinel token + a
SHA3‑512 **proof‑of‑work**): DeepAI's free tier is dramatically simpler because
it has **no real server‑side secret** in the anonymous path.

---

## Part B — The core client library

This is `server/deepai.js`. It (1) implements MD5, (2) reproduces
`generateIslandKey()`, and (3) sends the chat request and streams the reply.

Key pieces:

```js
const SALT = "hackers_become_a_little_stinkier_every_time_they_hack";
const USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/132.0.0.0 Safari/537.36";

const H = (s) => md5hex(s).split("").reverse().join("");   // reversed-MD5

function generateIslandKey(ua = USER_AGENT) {
  const rand = Math.round(Math.random() * 1e11) + "";
  return "tryit-" + rand + "-" +
    H(ua + H(ua + H(ua + rand + SALT)));
}
```

The send function builds the FormData and reads the streamed body:

```js
async function chat(messages, { model = "standard", onChunk } = {}) {
  const form = new FormData();
  form.append("chat_style", "chat");
  form.append("chatHistory", JSON.stringify(messages));
  form.append("model", model);
  form.append("session_uuid", crypto.randomUUID());
  form.append("sensitivity_request_id", crypto.randomUUID());
  form.append("hacker_is_stinky", "very_stinky");
  form.append("enabled_tools", "[]");

  const res = await fetch("https://api.deepai.org/hacking_is_a_serious_crime", {
    method: "POST",
    body: form,
    headers: {
      "api-key": generateIslandKey(),
      "User-Agent": USER_AGENT,
      Origin: "https://deepai.org",
      Referer: "https://deepai.org/",
    },
  });

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let full = "";
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    const chunk = decoder.decode(value, { stream: true });
    full += chunk;
    onChunk?.(chunk);
  }
  return full;
}
```

> The full file includes a complete, verified MD5 implementation and error
> handling (401 → "anonymous limit / paid‑only model"). See
> [`server/deepai.js`](../server/deepai.js).

**Free models (verified live):** `standard` (fast), `online` (web‑aware).
`genius` / `supergenius` return `401 "Only paid accounts can use genius"`.

---

## Part C — The backend server

Why a backend at all? Three reasons:
1. **CORS** — the browser can't call `api.deepai.org` directly (cross‑origin).
2. **Hide the algorithm** from the public page if you wish.
3. **Add your own rate limiting / logging / system prompt.**

`server/index.js` is a **zero‑dependency** Node HTTP server that:
- serves the static frontend, and
- exposes `POST /api/chat` which proxies to DeepAI and re‑streams the reply as
  **Server‑Sent Events (SSE)**.

```js
// POST /api/chat  { messages, model }  →  SSE: data:{"delta":"..."} … data:{"done":true}
const send = (obj) => res.write(`data: ${JSON.stringify(obj)}\n\n`);
await chat(messages, { model, onChunk: (t) => send({ delta: t }) });
send({ done: true });
```

It also includes a tiny per‑IP rate limiter (20 req/min) so your demo stays a
good citizen. Full file: [`server/index.js`](../server/index.js).

---

## Part D — The frontend UI

`public/index.html` + `public/app.js` give you a clean dark chat interface with:
- streaming bubbles (text appears as it arrives),
- conversation memory (sends the full `messages[]` each turn),
- a model picker, auto‑growing input, Enter‑to‑send.

The streaming read loop on the client:

```js
const reader = resp.body.getReader();
const decoder = new TextDecoder();
let buf = "";
for (;;) {
  const { done, value } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });
  const lines = buf.split("\n"); buf = lines.pop();
  for (const line of lines) {
    if (!line.startsWith("data:")) continue;
    const data = JSON.parse(line.slice(5).trim());
    if (data.delta) { acc += data.delta; bubble.textContent = acc; }
  }
}
```

Full files: [`public/index.html`](../public/index.html),
[`public/app.js`](../public/app.js).

---

## Part E — The CLI bot

Prefer the terminal? `cli.js` is a multi‑turn streaming chatbot in ~40 lines:

```bash
node cli.js                       # interactive
node cli.js "what is 2+2?"        # one-shot
MODEL=online node cli.js          # pick a model
```

---

## Part F — Run it

**Requirements:** Node.js **18 or newer** (for built‑in `fetch`, `FormData`,
`crypto.randomUUID`). Check with `node -v`.

```bash
# 1. enter the project
cd deepai-chatbot

# 2. start the web app (no npm install needed — zero dependencies!)
node server/index.js

# 3. open in your browser
#    http://localhost:3000

# …or use the terminal version
node cli.js "Hello!"
```

That's the entire setup. No keys, no env vars, no `npm install`.

---

## Part G — Deploy it (free hosting)

Because it's a plain Node server with no build step, it runs almost anywhere.

### Render.com (free tier)
1. Push this folder to a GitHub repo.
2. New → **Web Service** → connect the repo.
3. Build command: *(none)* · Start command: `node server/index.js`.
4. Render sets `PORT` automatically — the server already reads `process.env.PORT`.

### Railway / Fly.io / Cyclic
Same idea: start command `node server/index.js`, no build step.

### Docker (anywhere)
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
EXPOSE 3000
CMD ["node", "server/index.js"]
```
```bash
docker build -t deepai-chatbot . && docker run -p 3000:3000 deepai-chatbot
```

> ⚠️ On shared hosting, **all users share the host's IP**, so DeepAI's per‑IP
> rate limit is hit faster. For a public deployment, keep your own rate limiter
> on and consider caching.

---

## Part H — Troubleshooting

| Symptom | Cause | Fix |
|--------|-------|-----|
| `401 Unauthorized` | UA mismatch, or anonymous limit reached, or paid‑only model | Ensure `User-Agent` header == the UA used in the key; wait for the limit to reset; use `standard`/`online` |
| `"Only paid accounts can use genius"` | chose a paid model | Use a free model |
| `fetch is not defined` | Node < 18 | Upgrade Node to 18+ |
| Empty / no reply | endpoint changed, or network block | Re‑inspect the page in DevTools; the salt/endpoint may have changed |
| Garbled output | wrong stream parsing | The body is plain text — append chunks directly, don't `JSON.parse` them |
| Works in CLI, fails in browser | CORS | Always go through your `/api/chat` proxy, never call `api.deepai.org` from the page |

### If DeepAI changes the algorithm
This is undocumented, so it *will* drift. To re‑reverse it:
1. DevTools → Network → send a message → find the `POST` to `api.deepai.org`.
2. Copy the new `api-key` value and the request fields.
3. Search **Sources** for `api-key` → find the new generator function.
4. Update `SALT`, `USER_AGENT`, the hash, or the field list in `deepai.js`.

---

## API reference

### `server/deepai.js`

#### `chat(messages, options) → Promise<string>`
| param | type | description |
|-------|------|-------------|
| `messages` | `Array<{role, content}>` | full conversation; `role` ∈ `user`/`assistant`/`system` |
| `options.model` | `string` | `"standard"` (default) or `"online"` |
| `options.onChunk` | `(text)=>void` | called for each streamed chunk |
| `options.signal` | `AbortSignal` | optional cancellation |
| **returns** | `string` | the complete reply |

#### `generateIslandKey(userAgent?) → string`
Returns a fresh valid `api-key`.

#### `FREE_MODELS → string[]`
`["standard", "online"]`.

### HTTP API (your server)

#### `POST /api/chat`
Request JSON:
```json
{ "messages": [{ "role": "user", "content": "Hi" }], "model": "standard" }
```
Response: `text/event-stream`
```
data: {"delta":"He"}
data: {"delta":"llo"}
data: {"done":true}
```
On error: `data: {"error":"..."}`.

---

## FAQ

**Is this really free?**
Yes — it uses DeepAI's anonymous free tier. You're rate‑limited per IP, but
there's no key or payment.

**Which model is best?**
`standard` is fast and good for chat. `online` can reference web content.
`genius`/`supergenius` are paid‑only.

**Can I add a system prompt / personality?**
Yes — prepend `{ role: "system", content: "You are a pirate…" }` to `messages`
(or inject it server‑side in `/api/chat`).

**Will this break?**
Possibly — it's an undocumented endpoint. If it does, follow
[*If DeepAI changes the algorithm*](#if-deepai-changes-the-algorithm).

**Can I use it commercially?**
Not recommended. For anything serious, use DeepAI's official paid API and stay
within their Terms of Service.

---

### Project structure
```
deepai-chatbot/
├── server/
│   ├── deepai.js     # reverse-engineered DeepAI client (MD5 + key + chat)
│   └── index.js      # zero-dependency HTTP server + SSE proxy
├── public/
│   ├── index.html    # chat UI
│   └── app.js        # frontend streaming logic
├── cli.js            # terminal chatbot
├── package.json
└── docs/
    └── GUIDE.md      # this file
```

*Built as an educational reverse‑engineering exercise. Be a good internet citizen.* 🛰️
