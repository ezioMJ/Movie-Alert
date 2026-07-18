# booking-watch

Get a **Telegram** ping the moment a specific **movie + theatre** opens booking
on BookMyShow / District. Runs itself every ~10 minutes on **GitHub Actions**,
so there's nothing to keep running on your own machine.

It works by polling a URL you choose and watching for the transition from
"not bookable" to "booking open" for your theatre. Nothing site-specific is
hardcoded, so if BookMyShow/District tweak their pages you just edit
`config.json` — not the code.

---

## 1. Create a Telegram bot (2 minutes)

1. In Telegram, message **@BotFather** → send `/newbot` → follow prompts.
2. It gives you a **bot token** like `123456:ABC-DEF...`. Save it.
3. Open a chat with your new bot and send it any message (e.g. `hi`). This is
   required before it can message you.
4. Get your **chat id**: open this URL in a browser (paste your token):
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```
   Look for `"chat":{"id":123456789` — that number is your `TELEGRAM_CHAT_ID`.

## 2. Find the URL to watch

You want the URL where showtimes for your movie appear. Two options:

**A. Simple (the showtimes page).** On BookMyShow, open the movie's
"Book tickets" page for your city and copy the address bar URL, e.g.
`https://in.bookmyshow.com/movies/<slug>/buytickets/<code>/<city>`.
This is the easiest and works for most cases.

**B. Most reliable (the internal API).** These sites load showtimes via a
background request. To capture it:
1. Open the showtimes page in Chrome, press **F12** → **Network** tab.
2. Filter by **Fetch/XHR**, reload the page.
3. Find the request whose response contains venue/showtime data (search the
   responses for your theatre's name).
4. Right-click it → **Copy** → **Copy as cURL**, and copy the request URL.
5. If it needs headers/cookies, add them as a JSON object in the
   `HEADERS_JSON` secret (see below).

> Note: these internal APIs aren't official and may change or require cookies.
> If option B stops working, fall back to option A. Keep the poll interval
> polite (the default 10 min is fine).

## 3. Configure

```bash
cp config.example.json config.json
```

Edit `config.json`:
- `movie`   — the movie title as it appears on the page (helps avoid false hits).
- `theatre` — the theatre name exactly as shown on the site.
- `target_url` — the URL from step 2.

Matching is case-insensitive and whitespace-tolerant. Booking counts as "open"
when the theatre name **and** a booking signal (`book tickets`, `showtime`, ...)
are both present, and it's not showing only `notify me` / `coming soon`.

## 4. Test locally (optional but recommended)

```bash
pip install -r requirements.txt

export TELEGRAM_BOT_TOKEN=123456:ABC...
export TELEGRAM_CHAT_ID=123456789
python poller.py
```

It prints `available=True/False`. To confirm Telegram works, temporarily point
`target_url` at a page you know lists the theatre with booking open — you
should get a message.

## 5. Deploy on GitHub Actions

1. Create a **new GitHub repo** and push these files to it:
   ```bash
   git init && git add . && git commit -m "init booking-watch"
   git branch -M main
   git remote add origin https://github.com/<you>/booking-watch.git
   git push -u origin main
   ```
2. In the repo: **Settings → Secrets and variables → Actions → New repository secret**
   and add:
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_CHAT_ID`
   - `HEADERS_JSON` *(optional — only for API URLs that need headers/cookies)*
3. The workflow (`.github/workflows/booking-watch.yml`) runs every 10 minutes
   automatically. Trigger a test run anytime from the **Actions** tab →
   *booking-watch* → **Run workflow**.

When booking opens you get one Telegram message. State is saved in `state.json`
(committed back by the workflow) so you aren't re-notified every run.

### Public vs private repo

- **Public repo:** GitHub Actions minutes are **unlimited & free**. Recommended.
  Nothing sensitive is exposed — your token/chat id live in encrypted Secrets,
  only the movie/theatre/URL are in `config.json`.
- **Private repo:** the free tier gives ~2000 Actions minutes/month. Running
  every 10 min (~4300 runs/mo) would exceed it — bump the cron in the workflow
  to `*/30 * * * *` (every 30 min) to stay comfortably under.

## Adjusting

- **Interval:** edit the `cron` line in `.github/workflows/booking-watch.yml`
  (`*/10 * * * *` = every 10 min; GitHub's minimum is 5 and runs may lag a few
  minutes).
- **Multiple movies/theatres:** duplicate the folder into separate repos, or
  extend `config.json` into a list and loop in `poller.py` (ask and I'll wire
  it up).
- **False positives/negatives:** tune `open_signals` / `closed_signals` in
  `config.json`.

## Use it for your own movie / theatre (fork)

Anyone can run their own watcher — no need to touch this copy. Each fork has its
own secrets and its own Telegram chat, fully independent.

1. **Fork** this repo (top-right **Fork** button) into your own account.
2. **Enable Actions on the fork.** Forks have Actions disabled by default — open
   the **Actions** tab and click **"I understand my workflows, enable them."**
   Scheduled runs only fire from your fork's **default branch** (`main`).
3. **Add your own secrets** — *Settings → Secrets and variables → Actions*:
   - `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` (your bot — see step 1 above)
   - `SCRAPERAPI_KEY` (needed so the US-based runner can reach BookMyShow via an
     India IP — see the geo-block note below)
4. **Edit `config.json`** for your movie/date/theatre (see the config reference
   below), then **reset `state.json`** to `{"available": false}`.
5. Push, and trigger one manual run to confirm it prints `available=False`.

> No branches needed. "Date-only" vs "theatre-specific" is just the `detector`
> field in `config.json`, not different code — so a fork + config edit covers
> every use case. If you want to watch several targets at once, ask about the
> config-matrix setup instead of forking multiple times.

## Config reference — the three detectors

`config.json` picks a `detector`. All three build the page URL from
`url_template` + `requested_date` (the BMS date lives in the URL).

**`venue_date`** — alert when a *specific theatre* opens for a *specific date*.
The most precise mode. Uses the per-venue booking link `/<venueCode>/<date>`.
```json
{
  "detector": "venue_date",
  "movie": "The Odyssey (IMAX 2D)",
  "requested_date": "20260722",
  "venue_code": "INPR",
  "venue_label": "INOX: LUXE Phoenix Market City, Velachery",
  "url_template": "https://in.bookmyshow.com/movies/chennai/the-odyssey/buytickets/ET00480917/{date}"
}
```
Watch several theatres with `"venue_codes": ["PVPZ","INPR"]` (opens = any of them).

**`bms_date`** — alert when a date opens at *any* theatre. Uses date-dominance
(the requested date becomes the most-referenced date on the page).
```json
{ "detector": "bms_date", "requested_date": "20260722",
  "url_template": ".../buytickets/ET00480917/{date}", "min_references": 10 }
```

**`generic`** (default) — for non-date pages / other sites. Matches a `theatre`
string plus an open signal (`book tickets`, `showtime`) and no closed signal
(`notify me`, `coming soon`). Set `target_url` directly.

### How to find the values

- **`ET…` code + city + slug** — open the movie's "Book tickets" page on BMS and
  read them straight from the URL: `.../movies/<city>/<slug>/buytickets/<ETcode>/<date>`.
- **`requested_date`** — the date in `YYYYMMDD`, e.g. `20260722`.
- **`venue_code`** — fetch a date where the venue *is* open and look at its
  cinema link: `.../cinemas/<city>/<venue-slug>/buytickets/<CODE>/<date>` — the
  `<CODE>` (e.g. `PVPZ`, `INPR`) is the value. The bonus `curl … | grep -oE`
  command earlier prints the dates; grep the page for your theatre name to find
  its code.



## BookMyShow geo-block (why `SCRAPERAPI_KEY`)

BMS only serves India and blocks datacenter IPs. GitHub's runners are US-based,
so a direct request returns **403 Forbidden**. The poller routes through an
India IP when `SCRAPERAPI_KEY` is set (free tier from scraperapi.com), which
fixes it. Alternatives: set `PROXY_URL` to an India proxy, or run a self-hosted
runner on a machine physically in India. Running locally from India needs
neither. This is IP/geo-based — browser headers alone won't get past it.

## Files

| File | Purpose |
|------|---------|
| `poller.py` | Fetch URL, detect availability, send Telegram alert |
| `config.example.json` | Copy to `config.json` and fill in |
| `.github/workflows/booking-watch.yml` | Scheduled always-on runner |
| `requirements.txt` | Python deps (`requests`) |
| `state.json` | Auto-managed; tracks last-seen availability |
