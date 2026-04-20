# Auras — Production landing page

Static HTML landing page + one Vercel serverless function that proxies live chat to Anthropic's Claude API. Your API key stays server-side, rate limits are enforced, and you can swap the demo persona without touching any backend.

---

## What's in here

```
Auras Production/
├── index.html          ← the landing page (single self-contained file)
├── api/
│   └── chat.js         ← Vercel serverless fn — calls Anthropic
├── vercel.json         ← Vercel config
├── package.json        ← marks this as a Node project (needed for api/)
├── .gitignore
└── README.md           ← this file
```

The landing page talks to `/api/chat` on the same origin. No CORS, no public API key, no surprises.

---

## Deploy — the 5-minute version

### 1. Push to GitHub

Drop these files into your existing repo (or a new one), commit, and push. Vercel will auto-detect the `api/` folder as serverless functions.

```bash
git add .
git commit -m "Add Auras landing + serverless chat"
git push
```

### 2. Add your Anthropic API key to Vercel

In your Vercel dashboard:

1. Open the project → **Settings** → **Environment Variables**
2. Add a new variable:
   - **Name:** `ANTHROPIC_API_KEY`
   - **Value:** `sk-ant-…` (your key)
   - **Environments:** Production, Preview, Development (all three)
3. Save, then **redeploy** (Deployments → latest → ⋯ → Redeploy) so the new env var takes effect.

### 3. Set a hard spend cap in the Anthropic console

This is your safety net — non-negotiable.

1. Go to https://console.anthropic.com/ → **Settings** → **Billing** → **Limits**
2. Set a monthly spending limit (e.g. **$20 / ~R380**) on the API key you just pasted above.
3. If abuse ever gets past the other rate limits, the key silently stops working until next month. You can't overspend.

### 4. Done

Visit your domain. The landing page loads. The hero chat is live. You have your receipts.

---

## Rate limiting — how it works

Three layers, strongest → weakest:

| Layer | Where | Limit | Bypass difficulty |
|---|---|---|---|
| **Anthropic spend cap** | Anthropic console | Whatever you set (e.g. $20/mo) | Impossible — hard ceiling |
| **Per-IP rate limit** | `api/chat.js` in-memory | 20 msgs / IP / hour | Requires rotating IPs / VPNs |
| **Per-visitor cap** | `localStorage` in browser | 10 msgs / visitor / day | Trivial (clear storage / incognito) |

When any limit hits, the chat shows a friendly "you've hit the demo limit — get your own Auras for R299/mo" nudge that converts instead of frustrates.

### Tuning the limits

Edit the constants at the top of `api/chat.js`:

```js
const PER_IP_LIMIT      = 20;                // messages per IP ...
const PER_IP_WINDOW_MS  = 60 * 60 * 1000;    // ... per rolling hour
const MAX_TOKENS        = 300;               // max response length
```

For the per-visitor cap, edit `index.html` — search for `PER_VISITOR_CAP = 10`.

### Upgrading to hard rate limits (optional)

The in-memory IP limiter resets when Vercel spins up a new serverless instance. For a lot of traffic, plug in **Vercel KV** or **Upstash Redis** — ~20 lines of code. Happy to wire that up when you need it.

---

## Cost napkin math

Claude Haiku 4.5 is currently **~$0.80 / 1M input tokens** and **~$4 / 1M output tokens**. For Auras' short-reply style:

- **~1,000 input tokens + ~100 output tokens per message** = ~$0.0012 per message (~R0.02)
- **1,000 messages / month** ≈ $1.20 (~R22)
- **10,000 messages / month** ≈ $12 (~R220)

At the default rate limits (20/IP/hr), realistic traffic is in the low-thousands at most. **Set your Anthropic cap at $20 and you genuinely cannot overspend.**

---

## Changing the demo persona

The hero chat and playground use the personas defined at the top of `index.html` — search for `const PERSONAS = {`. The hero uses the `auras` persona (Auras answering questions about Auras).

To change what it says:

1. Open `index.html`
2. Find `auras:` inside `PERSONAS`
3. Edit the `system:` prompt — this is what the AI knows about your business
4. Edit the `seed:` conversation — the pre-baked opening exchange
5. Edit the `suggestions:` — the tap-chips below the chat

Commit, push, Vercel redeploys automatically.

---

## Local development

Install the Vercel CLI once:

```bash
npm i -g vercel
```

Then in this folder:

```bash
vercel dev
```

That runs the static file *and* the serverless function locally at `http://localhost:3000`. Put your API key in a `.env.local` file in this folder:

```
ANTHROPIC_API_KEY=sk-ant-...
```

`.env.local` is gitignored — it won't get pushed.

---

## Editing the page

The whole site is one `index.html` file. Inside is standard React (loaded via CDN) and inline styles — no build step, no npm install for the frontend. Edit, save, refresh.

If you want to split it into proper components later, it's straightforward to pull the React code into separate files and add a build step. Say the word.

---

## Known quirks

- **First request after a deploy is slow** (~1–2s cold start). Subsequent requests are fast.
- **In-memory rate limit resets on new serverless instances.** Not a bug, just a limitation of stateless functions without a database.
- **The localStorage cap is per-browser, not per-person.** Someone can clear it and retry. That's fine — the IP limiter and Anthropic cap catch abuse.

---

Questions or broken things? The chat on the site goes to a real AI — ask it. 😏
