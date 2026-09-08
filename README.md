# SlabWala — ITR preparation assistant (FY 2025-26 / AY 2026-27)

A single self-contained HTML file. Open `itr-ledger.html` in a browser — no install, no build step, no server. Every file you upload is parsed **in your browser**; nothing is uploaded anywhere.

---

## Publishing this for other people to use

This is designed to be published. The architecture that makes that workable is
**bring-your-own-key**: the tool ships with no API key, and each visitor enters
their own. Their browser then talks straight to the data provider. Your server
(or GitHub Pages, or wherever it's hosted) never sees their key, their documents,
or their figures — because it never receives them. It's a static file.

That distinction is what keeps a public deployment on the right side of the data
licensing. Read the section below carefully, but the short version:

| What you do | Standing |
|---|---|
| Publish the tool, users bring their own keys | **Fine.** Each user's own personal use, under their own account |
| Publish it with **your** key embedded | **Don't.** Redistribution of data, plus your key gets abused and billed/blocked |
| Charge for it, or run it as a service for clients | **Commercial** — needs a paid licence from the provider |

**Never hard-code a key into a public deployment.** Beyond the licensing issue, a
key in a client-side file is visible to anyone who views source — it will be
scraped and your quota exhausted within days.

### Three things to be honest with your users about

If strangers are going to file real tax returns with this, these need to be
visible, not buried — all three are stated inside the tool itself, and you should
keep them that way if you modify it:

1. **The FX rate is not the one CBDT specifies.** ECB reference rates, not the SBI
   TT buying rate. Close, but not the prescribed figure. Users must be able to
   override — that's why every rate is editable.
2. **It is not a filing service.** Nothing is submitted. The output goes into the
   Department's own utility, which does the real validation.
3. **The keyless fallback uses an undocumented Yahoo endpoint.** It works and
   requires no setup, which is why it's the zero-config path. But it isn't a
   published API and may breach Yahoo's terms. Because calls are made client-side,
   they come from each user's own browser and IP rather than from your server —
   which limits your exposure, but doesn't make the endpoint any more official.
   Encouraging users onto a free key is the cleaner path, and the tool now says so
   plainly on the price-source panel.

### Practical notes for a public deployment

- **A user on the free Alpha Vantage tier gets ~8 builds/day** (25 requests, ~3 per
  ticker). Twelve Data's 800/day is far more forgiving — it's the default for that
  reason, and worth keeping as the default if you fork this.
- **No analytics, no tracking, no backend.** Worth saying out loud on your landing
  page: people are uploading Form 16 and broker statements, and "it never leaves
  your browser" is only credible if there's genuinely nothing to leave to. Keep it
  that way.
- **`LICENSE`** (MIT) covers the software. It explicitly does **not** grant rights
  to the market data — that's between each user and their provider.

---

## Do I need to buy an API licence?

**For preparing your own tax return: no. The free tiers cover it.**

Both providers explicitly permit personal / internal use on their free tier, which is exactly what filing your own return is:

- **Twelve Data** — "Individual plans are intended strictly for personal or internal use."
- **Alpha Vantage** — free tier is for "personal, non-commercial use."

You would need a paid licence in these cases:

| Situation | Licence needed? |
|---|---|
| Preparing your own return | **No** — free tier |
| A CA preparing returns for clients | **Yes** — that's commercial use; contact the provider |
| Distributing this tool with your key embedded | **Yes** — redistribution needs a separate agreement |
| Publishing/reselling the price data itself | **Yes** — redistribution agreement |

Two things worth knowing so you don't over-buy:

1. **You don't need real-time data.** The expensive tiers (Alpha Vantage from ~$99.99/mo) exist for *real-time US market data*, which is exchange-regulated. This tool only uses **historical end-of-day** prices for a past calendar year. That's the cheap/free part of every provider's catalogue.
2. **You don't need a key at all to start.** With no key the tool still works using keyless sources — it just falls back to a Yahoo endpoint that is undocumented and less dependable (see below).

---

## Rate limits

| Source | What it provides | Free limit | Key needed |
|---|---|---|---|
| **Twelve Data** *(console-only, advanced)* | Prices, dividends | **800 requests/day** | Yes (free) |
| **Alpha Vantage** *(console-only, advanced)* | Prices, dividends, company profile | **25 requests/day**, 5/min | Yes (free) |
| **Stooq** *(default)* | Prices | No published limit | No |
| **Yahoo chart endpoint** | Prices, dividends, company profile | No published limit | No |
| **Frankfurter (ECB)** | USD→INR rates | No published limit | No |

### What one run actually costs

About **3 requests per ticker**: one for prices, one for dividends, one for company details. Results are cached for the browser session, so rebuilding the same schedule repeatedly costs nothing extra.

- On **Twelve Data (800/day)** — effectively unlimited for this purpose.
- On **Alpha Vantage (25/day)** — roughly 8 full builds per day.

### Why Twelve Data is the default

Two reasons, both practical:

1. **800/day vs 25/day.**
2. **Alpha Vantage's free tier may cap price history at ~100 data points** even when full history is requested. That's about five months of trading days — *not enough to compute a full calendar-year peak value*, and nowhere near enough to reach vest dates from earlier years.

That second point is a correctness risk, not just an inconvenience, so the tool now **refuses a truncated price series** and falls through to another source rather than silently computing a peak from partial data. If you see a message about "refusing to compute a peak from a partial series," that guard is doing its job.

### About the keyless fallback

Stooq and Frankfurter are fine. The **Yahoo endpoint is undocumented** — it is not a published API, it changes without notice, and relying on it may breach Yahoo's terms. It works, and it's there so the tool functions with zero setup, but adding a free key moves you onto a documented API that officially supports browser use. The tool tells you which source served each of prices / dividends / company details, so you can always see what you're relying on.

---

## Exactly which endpoints are called

One click of **Fetch everything** makes **3 requests** per ticker — one per distinct
data type. They cannot be collapsed into a single HTTP call because prices,
dividends and company profile are separate endpoints on every provider; what has
been eliminated is duplicate calls for the *same* data.

### With a Twelve Data key (default)

```
GET https://api.twelvedata.com/time_series
      ?symbol=CRM&interval=1day
      &start_date=<earliest vest date>&end_date=2025-12-31
      &outputsize=5000&apikey=<your key>

GET https://api.twelvedata.com/dividends
      ?symbol=CRM&start_date=2025-01-01&end_date=2025-12-31&apikey=<your key>
```

Twelve Data has no free company-profile endpoint, so the third call falls back to
Yahoo (below).

### With an Alpha Vantage key

```
GET https://www.alphavantage.co/query
      ?function=TIME_SERIES_DAILY&symbol=CRM&outputsize=full&apikey=<your key>

GET https://www.alphavantage.co/query
      ?function=DIVIDENDS&symbol=CRM&apikey=<your key>

GET https://www.alphavantage.co/query
      ?function=OVERVIEW&symbol=CRM&apikey=<your key>
```

### With no key at all (default for a new visitor)

```
GET https://stooq.com/q/d/l/?s=crm.us&d1=<from>&d2=<to>&i=d          # prices, CSV
GET https://query1.finance.yahoo.com/v8/finance/chart/CRM?...&events=div
GET https://query1.finance.yahoo.com/v10/finance/quoteSummary/CRM?modules=assetProfile,price
```

### Always called, regardless of key — the USD→INR rates

```
GET https://api.frankfurter.app/<from>..2025-12-31?from=USD&to=INR
```

Frankfurter serves European Central Bank reference rates. It is free, keyless, and
has no published rate limit. **It is not affected by your API key choice** — the
price providers supply share prices only, never the currency conversion.

## Configuration you need to do

**None.** It works on first open. There is no API key to enter anywhere in the UI —
the *Online price lookup* panel is a single on/off switch:

| | |
|---|---|
| **ON** (default) | Prices, USD rates, dividends and company details are fetched from free public sources, directly from the visitor's browser. No sign-up. |
| **OFF** | No network requests at all. The three valuation figures are typed in by hand. |

The one field the lookup genuinely cannot guess is the **ticker** (e.g. `CRM`) —
equity-plan statements list a company name, and price APIs only accept exchange
symbols.

### The dedicated Foreign Assets tab

The masthead has a **"Foreign Assets from E*TRADE / broker exports"** button that
opens a focused page for the single most common reason people open this tool:
upload your broker exports → fetch → build → download Schedule FA. No profile, no
situation checklist, nothing else in the return is touched. It reuses the exact
same builder and exports as the full wizard, so the two paths cannot produce
different numbers.

Raw broker exports (E*TRADE Holdings / Gains & Losses, Fidelity Activity,
StockPlan Connect Releases) are converted automatically into the lot format the
Table A3 builder needs — you no longer have to maintain a separate working sheet.
Initial value is `qty × vest FMV × vest-date USD rate`, so it shows as ₹0 until
you click *Fetch everything* and the vest-date rates arrive; that's deliberate
rather than fabricated.

### For self-hosters who want a documented API anyway

The keyed-provider code paths (Twelve Data, Alpha Vantage) still exist but have
no UI. If you host this and want them, set the key from the browser console:

```js
state.stockApiKey = 'your-key'; state.stockApiProvider = 'twelvedata';
```

This is deliberately not a UI feature: asking an ordinary taxpayer to register
for a financial-data API before filing is not a reasonable ask, and a key
embedded in a public page would be scraped within days.

## Subtle details worth knowing

- **Two different dates for the closing value.** The share price is the last close
  *on or before* 31 December; the USD rate is the rate *for* 31 December. These are
  resolved independently, because exchanges and rate publishers observe different
  holidays. Pinning both to the same date silently produces a wrong closing figure.
- **Peak is a daily maximum, not a maximum of maxima.** It is the highest
  `price × that same day's rate` across the year — *not* the year's highest price
  multiplied by the year's highest rate. Those differ whenever the two highs fall on
  different days, and the naive version overstates the peak.
- **Dividends are counted per lot, by payment date.** A lot only collects dividends
  paid on or after its own vest date, and each payment converts at that payment
  date's rate. Alpha Vantage's dividend feed exposes both an ex-date and a pay date;
  the pay date is used, which is the correct basis for "amount paid/credited during
  the period".
- **The fetch window starts at your earliest vest date, not 1 January.** Vest-date
  prices and rates are needed for the initial value, and those can be years earlier.
  But peak and closing are still computed strictly within the reporting calendar
  year, so an earlier year's high can never leak into the reported peak.
- **A truncated price series is refused, not used.** If a provider returns less
  history than the window needs, the response is rejected and the next provider is
  tried — a partial series would still "work" and quietly produce a wrong peak.
- **Caching is range-aware and per session.** Fetching a wider window later reuses
  the wider result; nothing is stored between page loads.
- **Nothing is batched across tickers.** With multiple distinct tickers the cost is
  3 requests each. Twelve Data does accept comma-separated symbols on some
  endpoints, which would be the route if you ever needed to optimise that.

## Known limitations — read before filing

- **FX is not the SBI TT buying rate.** CBDT specifies the SBI TTBR for Schedule FA. This tool uses ECB reference rates (via Frankfurter) because SBI's historical rates aren't available programmatically. The figures track closely but are **not** the prescribed rate. Every rate is editable — override them for a filing you're relying on.
- **Only Schedule CG is JSON-schema-verified.** Field names for `ScheduleCGFor23` were checked against the Department's published ITR-2 AY2026-27 schema. Schedule FA is verified against the official **CSV templates** (byte-for-byte) but not the JSON property names. Schedule BP is neither. Each page states its own verification status.
- **`Schedule112A` scrip-wise detail is not produced.** It requires a per-scrip sale-value / cost-acquisition / FMV breakdown for grandfathering; this tool only has the net gain your broker already computed. The LTCG total is correct, it just isn't decomposed into the Department's scrip-wise structure.
- **Not a filing tool.** Nothing is submitted anywhere. Output is a preparation aid — import into the official Offline Utility, verify every field, then file.
- **Audit-applicability flags are rules of thumb**, not determinations. F&O audit status also depends on your 44AD election history, which isn't tracked here.

---

## Exports

| Format | Covers |
|---|---|
| **Excel (.xlsx)** | Schedule FA (A1/A2/A3, one sheet each + a provenance sheet); Capital Gains (transactions + summary) |
| **CSV** | A1, A2, A3 individually — headers byte-identical to the official templates |
| **JSON** | Schedule CG, Schedule BP, Schedule FA individually, plus a combined return package |

The Excel export includes a **Notes & Provenance** sheet recording which dates, prices and FX rates each figure was built from, and which data sources served them — so the workbook still makes sense months later without needing to reconstruct the reasoning.

---

## Deployment

The repo now ships everything both platforms need — no manual dashboard
configuration required.

```
.
├── index.html                        # the app — Vercel/Pages serve this at "/"
├── itr-ledger.html                   # identical copy, kept for a stable direct link
├── vercel.json                       # static-hosting config for Vercel
├── .github/workflows/deploy-pages.yml# auto-deploys to GitHub Pages on every push
├── .gitignore
├── LICENSE
└── README.md
```

`index.html` and `itr-ledger.html` are byte-identical — `index.html` exists so
both platforms serve the app at the site root with zero configuration;
`itr-ledger.html` stays so any link you've already shared keeps working. If you
edit the app, save both (or just copy one over the other) so they don't drift.

### Get it onto GitHub

```bash
git init
git add .
git commit -m "SlabWala ITR assistant"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

### Deploy to Vercel

**Dashboard (no CLI needed):** [vercel.com/new](https://vercel.com/new) → Import
your GitHub repo → leave every build setting blank (Framework Preset: **Other**,
no build command, no output directory) → Deploy. `vercel.json` in the repo
handles the rest.

**CLI, if you prefer:**
```bash
npm i -g vercel
vercel          # first deploy, follow the prompts
vercel --prod   # promote to production
```

Every subsequent `git push` to `main` redeploys automatically once the GitHub
repo is linked, whichever way you connected it.

### Deploy to GitHub Pages

One-time setup, then it's automatic:

1. Push the repo (above).
2. On GitHub: **Settings → Pages → Source → GitHub Actions**. (Just this
   dropdown — not "Deploy from a branch".)
3. That's it. `.github/workflows/deploy-pages.yml` is already in the repo and
   runs on every push to `main`, publishing to
   `https://<you>.github.io/<repo>/`.
4. Watch progress under the repo's **Actions** tab. First run takes a minute or
   two; Pages needs the workflow to run once before the URL goes live.

To trigger a deploy without a new commit (e.g. right after enabling Pages), use
**Actions → Deploy to GitHub Pages → Run workflow**.

### Which one should I actually use?

Either is fine — it's a static file with no backend, so there's no functional
difference. Rough guide: **Vercel** if you want a custom domain with less DNS
fuss and instant preview URLs on every branch/PR; **GitHub Pages** if you want
one thing hosted directly from the repo with nothing else to sign up for.
Nothing stops you running both from the same repo at once, which is exactly
what this setup does.

### Deployment checklist

- [ ] No API key hard-coded anywhere in the file (check before every push)
- [ ] `index.html` and `itr-ledger.html` match (`diff` them, or just re-copy)
- [ ] `LICENSE` and `README.md` included, so users can see the terms and limits
- [ ] Landing page states: not tax advice, not a filing service, verify before filing
- [ ] If you add analytics later, say so — the privacy claim is a big part of why
      anyone would upload a Form 16 to this

Keys are entered at runtime and held in browser memory only; they are never
written into the file, so the deployed copy carries no secret of yours.

---

*Not affiliated with the Income Tax Department of India. Verify everything before filing.*
