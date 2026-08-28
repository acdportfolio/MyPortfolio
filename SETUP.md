# Chatbot + Notification System — Setup Guide

## How it's designed to keep your credentials secure

```
Browser (chat widget in index.html)
        │  fetch() — only your message text, nothing secret
        ▼
Cloudflare Worker (this folder)          ← your Anthropic API key and Discord
        │           │                      webhook URL live here ONLY, as
        │           │                      encrypted secrets on Cloudflare's
        │           ▼                      servers. They are never written to
        │     Discord webhook               index.html, never committed to
        │     (visitor notification)        GitHub, never sent to the browser.
        ▼
  Anthropic API (chat reply)
```

Your GitHub Pages site stays 100% static — it only ever talks to your own Worker,
and the Worker is the only thing that holds real credentials.

## 1. Get the two credentials you'll need

- **Anthropic API key** — console.anthropic.com → API Keys → Create Key
- **Discord webhook URL** — create a private Discord server (or just a channel in an
  existing one) → Channel Settings → Integrations → Webhooks → New Webhook → Copy URL

  (Prefer Telegram or email instead of Discord for notifications? Tell me and I'll swap
  the `notifyDiscord` call for a Telegram Bot API call or an email API — same pattern,
  same security model.)

Keep both of these out of chat, docs, and commit messages — you'll paste them directly
into the `wrangler secret put` prompts below, which send them straight to Cloudflare over
an authenticated, encrypted connection.

## 2. Install and log in to Wrangler (Cloudflare's CLI)

```bash
npm install -g wrangler
wrangler login
```

This opens a browser window to authorize the CLI against your (free) Cloudflare account.

## 3. Configure and deploy

Edit `wrangler.toml` in this folder and set `ALLOWED_ORIGIN` to your real GitHub Pages
URL, e.g. `https://yourusername.github.io` (no trailing slash).

Then from inside this `worker/` folder:

```bash
wrangler deploy
```

Wrangler will print a URL that looks like:

```
https://portfolio-api.yoursubdomain.workers.dev
```

That's your Worker's public address — copy it.

## 4. Set your secrets (never typed into index.html or this repo)

```bash
wrangler secret put ANTHROPIC_API_KEY
wrangler secret put DISCORD_WEBHOOK_URL
```

Each command will prompt you to paste the value — it's uploaded directly to Cloudflare
and stored encrypted. It won't appear in `wrangler.toml`, your terminal history file,
or anywhere in the repo.

## 5. Wire the Worker URL into your portfolio

In `index.html`, there are two placeholders to replace with the URL from step 3:

1. Near the bottom, in the chatbot `<script>` block:
   ```js
   const CHAT_API = "https://portfolio-api.YOUR-SUBDOMAIN.workers.dev";
   ```
2. In the `<meta http-equiv="Content-Security-Policy">` tag near the top of `<head>`,
   inside `connect-src`:
   ```
   connect-src 'self' https://portfolio-api.YOUR-SUBDOMAIN.workers.dev;
   ```

Both must point at the exact same Worker URL, or the browser's CSP will silently block
the chat requests.

## 6. Test it

Open your site, click the chat bubble bottom-right, send a message, and confirm:
- you get a reply in the widget, and
- a message shows up in your Discord channel.

Try "Leave your email" too and confirm a second Discord message arrives.

## Cost and abuse protection

- The Worker uses **Claude Haiku** by default — cheap and fast, appropriate for a
  portfolio FAQ bot. You can change the `model` field in `worker.js` if you'd prefer
  a different one.
- A basic per-IP rate limit (8 chat messages / minute, 4 contact submissions / minute)
  is built into `worker.js`.
- For stronger protection against sustained abuse, add a free **Rate Limiting rule**
  in the Cloudflare dashboard: your zone → Security → WAF → Rate limiting rules →
  match path `/api/*`.
- The Worker only accepts requests whose `Origin` header matches `ALLOWED_ORIGIN`
  exactly, so it can't be called from other sites even if someone finds the URL.

## Updating later

Change anything in `worker.js` or `wrangler.toml`, then just re-run:

```bash
wrangler deploy
```

Your secrets stay in place across deploys — you only need to re-run `secret put` if
you're rotating a key.
