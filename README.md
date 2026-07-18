# Movie-Alert

Get a **Telegram** ping the moment a specific **movie + theatre + date** opens
for booking on BookMyShow. Runs on **GitHub Actions**, triggered every ~10 min
by **cron-job.org** — nothing to keep running on your own machine.

## How it works

1. **cron-job.org** triggers the workflow every ~10 min (GitHub's own scheduler
   is unreliable, so we trigger it externally).
2. `poller.py` fetches the BMS page through **ScraperAPI** (an India IP — BMS
   blocks foreign/datacenter IPs and returns 403 otherwise).
3. It checks whether your target is open and, on the `False → True` flip, sends
   one Telegram message. Last-seen state lives in `state.json`.

## 1. Telegram bot

- Message **@BotFather** → `/newbot` → copy the **token**.
- Send your new bot any message, then open
  `https://api.telegram.org/bot<TOKEN>/getUpdates` and copy the `chat.id` — that's
  your **chat id**.

## 2. ScraperAPI key

Sign up at **scraperapi.com** (free tier) and copy your API key.

## 3. Add repo secrets

**Settings → Secrets and variables → Actions:**

- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_CHAT_ID`
- `SCRAPERAPI_KEY`

## 4. Configure `config.json`

Pick a `detector`. The page URL is built from `url_template` + `requested_date`.
After editing, reset `state.json` to `{"available": false}`.

**`venue_date`** — a specific theatre opens for a specific date (most precise):
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
Use `"venue_codes": ["PVPZ","INPR"]` to fire when *any* of several theatres open.

**`bms_date`** — a date opens at *any* theatre (date-dominance on the page):
```json
{ "detector": "bms_date", "requested_date": "20260722",
  "url_template": ".../buytickets/ET00480917/{date}", "min_references": 10 }
```

**Finding the values:** read `<city>/<slug>/<ETcode>/<date>` from the movie's
"Book tickets" URL. For `venue_code`, open a date where the theatre *is* open and
read its cinema link `.../cinemas/<city>/<venue-slug>/buytickets/<CODE>/<date>` —
the `<CODE>` (e.g. `PVPZ`, `INPR`) is the value.

## 5. Schedule it with cron-job.org

**a. GitHub token** — *Settings → Developer settings → Personal access tokens →
Fine-grained tokens → Generate*. Scope it to this repo, permission
**Actions: Read and write**. Copy the token.

**b. cron-job.org** — create a cronjob:

| Field | Value |
|---|---|
| URL | `https://api.github.com/repos/<you>/<repo>/actions/workflows/booking-watch.yml/dispatches` |
| Schedule | every 10 minutes |
| Method | `POST` |
| Header | `Accept: application/vnd.github+json` |
| Header | `Authorization: Bearer <your-token>` |
| Header | `X-GitHub-Api-Version: 2022-11-28` |
| Body | `{"ref":"main"}` |

Save, then **Run now**. GitHub returns `204`; a run appears under **Actions**.
From then on it fires every 10 min. Keep the token only in cron-job.org.

## Geo-block (why ScraperAPI)

BMS only serves India and blocks datacenter IPs, so GitHub's US runners get a
**403** on a direct request. `SCRAPERAPI_KEY` routes through an India IP and fixes
it. Alternatives: set a `PROXY_URL` secret (India proxy), or run from a machine in
India. It's IP/geo-based — headers alone won't get past it.

## Reuse it (fork)

Fork the repo, **enable Actions** on the fork (off by default), add your own
secrets, edit `config.json`, reset `state.json`, and set up your own cron-job.org
trigger. Each fork is independent with its own Telegram chat. No branches needed —
date-only vs theatre-specific is just the `detector` field.

## Files

| File | Purpose |
|------|---------|
| `poller.py` | Fetch page, detect availability, send Telegram alert |
| `config.json` | Your movie / date / theatre target |
| `.github/workflows/booking-watch.yml` | The runner (dispatched by cron-job.org) |
| `requirements.txt` | Python deps (`requests`) |
| `state.json` | Auto-managed; tracks last-seen availability |
