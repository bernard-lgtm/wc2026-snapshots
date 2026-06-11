# World Cup 2026 snapshots — self-hosted on GitHub (no dev, no server)

This folder is a complete, self-contained website **plus** the live data feed
for it. You can put the whole thing on your own GitHub account and it will show
live SportMonks data — with **no separate server and no developer needed.**

## How it works (and why it's set up this way)

SportMonks blocks direct calls from a reader's browser (no CORS), so the page
can't fetch it itself. Instead:

```
  GitHub Action (every ~5 min)                 GitHub Pages
  ── holds the token as a secret               ── serves index.html
  ── calls SportMonks (server-side)    ───►    ── serves data/fixtures.json
  ── writes data/fixtures.json                 the page reads that JSON (same
                                               origin, so no CORS problem)
```

The **GitHub Action is the "proxy."** It runs on GitHub's servers, where calling
SportMonks is allowed and the token stays hidden. The page just reads a static
JSON file sitting next to it. The token **never** reaches anyone's browser.

> **One honest tradeoff:** GitHub's scheduled Actions run **about every 5
> minutes** (sometimes a little slower under load) — not every few seconds. For
> these snapshot cards (today's fixtures, yesterday's results, standings,
> leaders) that's perfect. If you later want a true second-by-second *live
> score* ticker, that needs an always-on server (a free Cloudflare Worker does
> it — see `../wc-proxy/`). The page already falls back to its built-in demo
> snapshot any time the feed is missing, so it never breaks.

## What's in this folder

```
index.html                         the snapshot page (everything inlined: fonts, CSS, JS)
data/fixtures.json                 the live feed (a seed file; the Action overwrites it)
scripts/fetch-feed.mjs             the feed builder (calls SportMonks server-side)
.github/workflows/update-wc-feed.yml   the schedule that runs the builder
```

---

## Set it up — about 5 minutes, all in the GitHub website

### 1. Create a repository
- GitHub → **New repository**. Name it e.g. `wc2026-snapshots`.
- **Make it Public.** This gives you free, unlimited Actions minutes and free
  Pages. It's safe: the API token is stored as an encrypted *secret*, **not** in
  the files, and the fixtures JSON isn't sensitive. (Private also works but needs
  a paid plan for Pages and has a monthly Actions-minutes cap.)

### 2. Upload these files
- On the repo page: **Add file → Upload files**, then drag in **everything in
  this folder** (keep the folder structure).
- The `.github` folder can be hidden by your operating system. If the workflow
  didn't upload, add it by hand: **Add file → Create new file**, type the name
  exactly as
  `.github/workflows/update-wc-feed.yml`
  and paste in the contents of that file from this folder. Commit.

### 3. Add the SportMonks token as a secret
- Repo **Settings → Secrets and variables → Actions → New repository secret**.
- **Name:** `WC_API_TOKEN`
- **Value:** your SportMonks token
  (`ptQCnyko7MtWCpe6EdxUo7SlrG1PwCKPqbHCaZGqKbueGC30kj2MgX4hbFRK`)
- Save. This is the only place the token lives — encrypted, never shown again,
  never in the page.

### 4. Turn on GitHub Pages
- Repo **Settings → Pages**.
- **Source:** *Deploy from a branch* → Branch: **main** → Folder: **/ (root)** →
  **Save**.
- After a minute Pages gives you a URL like
  `https://<your-username>.github.io/wc2026-snapshots/`.

### 5. Run the feed once (don't wait for the timer)
- Repo **Actions** tab → if prompted, click to **enable workflows**.
- Pick **Update World Cup feed** → **Run workflow** → **Run**.
- It calls SportMonks, writes `data/fixtures.json`, and commits it. After it
  finishes (and Pages redeploys, ~1 min), open your Pages URL: the pill flips
  from **Demo data** to **Live data** and real fixtures appear.

That's everything. From now on the Action refreshes the feed every ~5 minutes on
its own.

---

## Good to know

- **Stopping it after the tournament:** Actions tab → Update World Cup feed →
  **⋯ → Disable workflow**. (Re-enable any time.)
- **It never shows a broken page:** if SportMonks is down or rate-limited, the
  builder fails *without* overwriting the last good `data/fixtures.json`, and the
  page falls back to the bundled demo snapshot regardless.
- **Group standings (settled):** live standings during play need nothing extra.
  The fully-settled season table needs SportMonks' final-tournament *Season ID*
  — confirm it with SportMonks if/when you wire the standings endpoint.
- **Editing the page later:** `index.html` here is a compiled bundle — don't edit
  it directly. Edit the source in the main project
  (`World Cup Snapshots - DEPLOY.html` + `snapshots.deploy.js`) and re-export.
- **Refresh cadence:** to poll faster than 5 min you'd move to an always-on host
  (Cloudflare Worker in `../wc-proxy/`), which is still free and still no-dev to
  deploy, just not GitHub-only.
