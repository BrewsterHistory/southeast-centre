# Southeast Centre — Chat Proxy Setup & Maintenance

The "Ask a Question" chat tab on the public site is powered by a small
Cloudflare Worker that holds your Anthropic API key and the research corpus
privately. The browser only ever talks to the Worker — never directly to
Anthropic — so the key is never exposed and the full research dump is not
sitting in your site's source code for anyone to copy in one shot.

This document covers:

1. First-time deployment (one-time, ~15 minutes)
2. Updating the research corpus (recurring, under a minute)
3. Keeping your local "source of truth" copy out of the public repo
4. Troubleshooting

---

## What's in this folder

- **`worker.js`** — the Cloudflare Worker code. Contains the entire research
  corpus baked in as a `SYSTEM` constant near the top. Lives privately on
  Cloudflare's edge; never served by GitHub Pages.
- **`index.html`** — the updated app file. The only chat-related change
  versus the previous version: it calls a `WORKER_URL` instead of Anthropic,
  and the API-key prompt UI is gone. Otherwise identical to your prior
  `index.html`.
- **`UPDATING.md`** — this file.

The previous `knowledge-base.js` file is **not** in this folder and should
**not** be in the public repo anymore. Keep your local working copy of it,
but stop committing it. Instructions below.

---

## 1. First-time deployment

### Step 1 — Cloudflare account (free, 3 min)

1. Go to **https://dash.cloudflare.com/sign-up** and create a free account.
   No credit card required.
2. Verify your email.

### Step 2 — Create the Worker (3 min)

1. In the Cloudflare dashboard, left sidebar: **Workers & Pages**.
2. Click **Create** → **Create Worker**.
3. Name it something like `southeast-chat`. Note the URL it shows you
   (e.g. `https://southeast-chat.YOUR-SUBDOMAIN.workers.dev`) — you'll
   need it later.
4. Click **Deploy** to create the placeholder, then **Edit code**.
5. **Delete everything** in the editor pane and paste the entire contents
   of `worker.js` from this folder.
6. Click **Save and deploy**.

### Step 3 — Set the Anthropic API key as a secret (2 min)

1. Still in your Worker, go to **Settings** → **Variables and Secrets**.
2. Click **Add variable**.
3. Switch the type to **Secret**.
4. Name it exactly: `ANTHROPIC_API_KEY`
5. Paste your real Anthropic key (`sk-ant-...`) into the value field.
6. Click **Deploy**.

The key is now encrypted at rest on Cloudflare. Even you can't view it again
through the dashboard — only the Worker can read it at runtime.

### Step 4 — Create the rate-limit storage (3 min)

1. Left sidebar: **Storage & Databases** → **KV**.
2. Click **Create namespace**.
3. Name it exactly: `RATE_LIMIT`
4. Click **Add**.
5. Go back to **Workers & Pages** → click your `southeast-chat` Worker.
6. **Settings** tab → scroll to **Bindings** → **Add binding**.
7. Choose **KV namespace**.
8. **Variable name**: `RATE_LIMIT`
9. **KV namespace**: select the `RATE_LIMIT` one you just created.
10. Click **Deploy**.

### Step 5 — Wire the Worker URL into the app (1 min)

1. Open `index.html` in a text editor.
2. Find this line (search for `WORKER_URL`):
   ```js
   const WORKER_URL = 'https://southeast-chat.YOUR-SUBDOMAIN.workers.dev';
   ```
3. Replace the placeholder URL with your actual Worker URL from Step 2.
4. Save the file.
5. Commit and push to your `southeast-centre` repository as you normally do.

### Step 6 — Set a hard spend cap on Anthropic (2 min)

1. Go to **https://console.anthropic.com**.
2. **Settings** → **Limits** (or **Usage limits**).
3. Set a monthly spend cap. $10 is a reasonable starting point — covers
   ~800 questions on Haiku 4.5 pricing. You can raise it later if you see
   real usage approaching the cap.
4. Save.

This is your last-line backstop: even if every other guardrail fails,
Anthropic itself will stop processing requests once you hit this cap.
No surprise bills are possible.

### Step 7 — Test it (2 min)

1. Wait ~2 minutes for GitHub Pages to rebuild after your push.
2. Visit `https://brewsterhistory.github.io/southeast-centre/`.
3. Click **Ask a Question**.
4. Type a real question (e.g. "Who owned Stonehenge?") and hit Send.
5. You should get a streamed answer in a few seconds.

If it works, you're done. If you get an error, see Troubleshooting below.

---

## 2. Updating the research corpus

When you finish a research session and want the chat to know about new
findings:

1. **Edit your local copy** of the research text. Recommended location and
   filename for clarity: `private/knowledge-base.txt` (a folder outside
   your repo, or a folder ignored by git — see Section 3 below).

2. **Open your Worker in the Cloudflare dashboard**:
   `dash.cloudflare.com` → Workers & Pages → `southeast-chat` → **Edit code**.

3. **In the editor**, find the line that reads `const SYSTEM = ` followed
   by a backtick. The research text starts there and runs until a closing
   backtick + semicolon (look for `` `; `` near the end of the SYSTEM
   block).

4. **Select all the text between the two backticks** and replace it with
   your updated research text.

5. **Click "Save and deploy"**. Takes about 5 seconds. Live immediately.

That's the entire workflow. No GitHub commit, no `index.html` change,
no anything else needed for a research-only update. The lot HTML files,
the photos, and the rest of the public site stay exactly as they are.

---

## 3. Keep your local research copy out of the public repo

Your old `knowledge-base.js` lived in the GitHub repo, which means anyone
visiting the live site could see it via View Source. With the proxy
architecture, that's no longer needed (the canonical research corpus
lives in the Worker). To remove it cleanly:

### One-time cleanup

```bash
# In your local clone of southeast-centre/
git rm --cached knowledge-base.js
echo "knowledge-base.js" >> .gitignore
echo "private/" >> .gitignore
git add .gitignore
git commit -m "Remove knowledge-base.js from repo; chat now uses Cloudflare Worker proxy"
git push
```

The `git rm --cached` removes the file from version control while leaving
your local copy intact. The `.gitignore` lines ensure neither the file
nor a `private/` folder ever gets committed again.

### Going forward

Keep your local source-of-truth copy somewhere convenient — either right
where it was on your filesystem, or moved to a `private/` folder under
your repo. Either way, git will ignore it.

When you research-update: edit local copy → paste into Worker → deploy.
The local file is the only "permanent" copy you control directly.

---

## 4. Troubleshooting

### "Something went wrong: Forbidden origin"

The Worker is rejecting your origin. Check that your site is actually
served from `https://brewsterhistory.github.io` (note: HTTPS). If you're
testing locally and you're not on `localhost:8000` or `127.0.0.1:8000`,
add your test origin to the `ALLOWED_ORIGINS` array near the top of
`worker.js` and redeploy.

### "Something went wrong: Request failed (401)"

The Worker is reaching Anthropic but Anthropic is rejecting the API key.

- Confirm the secret is named **exactly** `ANTHROPIC_API_KEY` (case-
  sensitive, underscores not dashes).
- Confirm the key starts with `sk-ant-` and is currently valid (try
  it manually in the Anthropic console).

### "Something went wrong: Request failed (429)"

You've hit the per-IP rate limit (50/hour). This will clear at the top
of the next hour. If you want to raise the limit, edit
`RATE_LIMIT_PER_HOUR` in `worker.js` and redeploy.

### "Something went wrong: Rate limit exceeded"

Same as above — the Worker is the source of this message.

### Chat "thinking…" forever, no error

Check the browser DevTools network tab. Most common cause: typo in the
`WORKER_URL` constant in `index.html`. The URL should match exactly what
Cloudflare shows for your Worker (starts with `https://`, ends with
`.workers.dev`).

### "Cannot find module 'RATE_LIMIT'" or similar in Worker logs

The KV binding isn't wired up. Workers & Pages → your Worker → Settings →
Bindings → confirm there's a KV namespace binding named `RATE_LIMIT`.
The variable name is case-sensitive.

### Page loads but `Ask a Question` panel is blank or broken

`index.html` may still have the old `<script src="knowledge-base.js">`
line. Search the file for that string and remove it if present. Or, if
you removed it but `knowledge-base.js` was already cached by your
browser, do a hard refresh (Ctrl+Shift+R or Cmd+Shift+R).

---

## Cost expectations

Haiku 4.5 pricing: $1 per million input tokens, $5 per million output
tokens. Your system prompt is ~7,500 tokens. A typical Q&A:

- Input: ~7,500 (system) + ~50 (question) = 7,550 tokens × $1/M = **$0.0076**
- Output: ~400 tokens × $5/M = **$0.002**
- **Total: ~$0.01 per question.**

A $10 monthly cap covers about 1,000 questions. For a Putnam County local
history audience that's far more than realistic traffic — most months
will probably be a fraction of $1. Watch the Anthropic console for the
first few weeks to see what your real cost curve looks like.

---

## Architecture summary

```
       Public                          Private
  ┌─────────────────┐         ┌──────────────────────┐
  │ brewsterhistory │  POST   │ Cloudflare Worker    │
  │   .github.io    │ ──────▶ │ (your_subdomain      │
  │                 │  msgs   │  .workers.dev)       │
  │  index.html     │         │                      │
  │  lots/*.html    │ ◀────── │  • SYSTEM (research) │
  │  images/...     │ reply   │  • API key (secret)  │
  └─────────────────┘         │  • Rate limit (KV)   │
                              │  • CORS lock         │
                              └──────────┬───────────┘
                                         │
                                         │ uses key
                                         ▼
                              ┌──────────────────────┐
                              │   Anthropic API      │
                              │   (Haiku 4.5)        │
                              └──────────────────────┘
```

Visitors only ever see the left box. The middle box holds everything
sensitive. The right box is reached only by the middle box.
