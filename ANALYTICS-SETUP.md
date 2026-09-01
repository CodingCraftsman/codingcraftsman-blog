# Finishing the blog analytics setup (GoatCounter)

Task #320 wired cookieless, privacy-first page analytics into the blog. The code
is in place but **disabled** until you paste your GoatCounter code. Here's how to
finish — about 2 minutes.

## What's already done (in this working tree)

- `layouts/_partials/extend_head.html` — adds the GoatCounter beacon in `<head>`,
  gated on a config value. Emits **nothing** while the value is empty.
- `hugo.toml` — adds `params.goatcounterCode = ""` (the placeholder).

No theme files were touched (PaperMod stays a clean submodule), and the analytics
is a **no-op** until you set the code below.

## Step 1 — Create the free GoatCounter account (~1 min)

1. Go to <https://www.goatcounter.com/> and click **Sign up**.
2. Pick a **code** (subdomain) — this becomes `YOURCODE.goatcounter.com`.
   Keep the pen name intact: use something like **`guillermofaro`** or
   **`codingcraftsman`**, NOT your real name (the blog is anonymous).
3. For **Site domain**, enter `codingcraftsman.blog`.
4. Set your email + password, confirm. That's the whole account.

> Free for non-commercial use. No credit card. The dashboard lives at
> `https://YOURCODE.goatcounter.com`.

## Step 2 — Paste the code into `hugo.toml` (~30 sec)

Open `hugo.toml`, find:

```toml
  goatcounterCode = ""
```

Set it to **just the code** you chose (not the full URL):

```toml
  goatcounterCode = "guillermofaro"
```

That's it — the partial builds the full endpoint
(`https://guillermofaro.goatcounter.com/count`) for you.

## Step 3 — Ship it

Commit the changed files and push; the GitHub Pages Action deploys as usual:

```bash
git add hugo.toml layouts/_partials/extend_head.html ANALYTICS-SETUP.md
git commit -m "feat: add cookieless GoatCounter analytics (#320)"
git push
```

Within a minute of the deploy going live, load <https://codingcraftsman.blog>
in a browser and refresh once or twice.

## Step 4 — View your numbers

Go to `https://YOURCODE.goatcounter.com` and log in. You'll see:

- **Pageviews** over time (day/week/month)
- **Referrers** — where visitors came from (search, HN, links, etc.)
- **Top pages** — which posts get read
- Browser / OS / country breakdowns (all aggregate, no PII)

To stop counting your own visits: on the GoatCounter dashboard go to
**Settings → Ignore your own views** — it drops a local-only flag in your
browser (a per-browser dev convenience; real visitors still get no cookie).

## How to turn it off later

Set `goatcounterCode = ""` again and redeploy. Nothing is emitted; no leftover
cookies to clear (there were never any).

## Why this satisfies the "don't get in the way" constraints

- **Cookieless** — GoatCounter's `count.js` sets no cookies, so **no consent
  banner is required**.
- **Non-blocking** — the script is `async`; it never blocks first paint.
- **No layout shift** — it's an invisible beacon; zero CLS.
- **Tiny** — `count.js` is ~3.5 KB, served from GoatCounter's CDN.
- **Privacy-first** — no PII, no cross-site/fingerprint tracking, and it
  **honors Do-Not-Track** automatically.
