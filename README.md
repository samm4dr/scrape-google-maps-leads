# How to Scrape Google Maps into a Lead List in 2026 (No Code Required)

You need a list of plumbers in Manchester, dentists in Miami, or roofers in Lyon — with phones, emails, and websites. Doing it by hand takes days. Writing a scraper takes hours... until Google rate-limits you into CAPTCHA hell.

Here's the 2026 way: run a maintained **Apify actor** that handles proxies, pagination, and anti-blocking — then let your AI agent do the plumbing.

## The stack

- **[Apify](https://console.apify.com/sign-up?fpr=dx06p)** — actor marketplace with ready-made Google Maps scrapers. Free tier includes monthly credits — enough to test this end-to-end.
- **Your AI agent** (Claude Code, Cursor, Codex...) — runs the pipeline via a skill that knows the actor inputs.

No proxies to configure. No headless browser tuning.

## Step 1 — Get your API token

1. [Create an Apify account](https://console.apify.com/sign-up?fpr=dx06p) (free credits included)
2. Console → Settings → Integrations → copy your API token
3. `export APIFY_TOKEN=your_token`

## Step 2 — Run the lead-extraction actor

The `lukaskrivka/google-maps-with-contact-details` actor doesn't just grab businesses — it visits each website and extracts **emails and social profiles** automatically:

```bash
curl -X POST "https://api.apify.com/v2/acts/lukaskrivka~google-maps-with-contact-details/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "searchStringsArray": ["dentist"],
    "locationQuery": "Miami, FL",
    "maxCrawledPlacesPerSearch": 100,
    "language": "en"
  }'
```

You get back a JSON array with `title`, `phone`, `website`, `emails`, `address`, `totalScore`, `reviewsCount`, `categoryName`.

## Step 3 — Clean lead CSV with Python

```python
from apify_client import ApifyClient
import csv

client = ApifyClient("APIFY_TOKEN")
run = client.actor("lukaskrivka/google-maps-with-contact-details").call(run_input={
    "searchStringsArray": ["plumber"],
    "locationQuery": "Austin, TX",
    "maxCrawledPlacesPerSearch": 200,
})

with open("prospects.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["business", "phone", "email", "website", "rating", "reviews"])
    for p in client.dataset(run["defaultDatasetId"]).iterate_items():
        if p.get("phone") or p.get("emails"):
            w.writerow([
                p["title"], p.get("phone", ""),
                (p.get("emails") or [""])[0],
                p.get("website", ""), p.get("totalScore", ""),
                p.get("reviewsCount", ""),
            ])
```

Dedupe by `website` or `phone`, filter for `totalScore >= 3.5` — and you have a clean outreach list.

## Bonus: let your agent run it for you

I packaged this (plus 7 other scraping patterns — jobs, Amazon, social, SERP) as Agent Skills:

```bash
npx skills add g4dr/apify-skills
```

After installing, tell Claude Code *"get me all roofers in Birmingham"* — it picks the right actor, builds the run input, and hands you the data. Full repo: [g4dr/apify-skills](https://github.com/g4dr/apify-skills).

For the bigger picture (proxies, headless browsers, niche actors), I maintain an [awesome-web-scraping](https://github.com/g4dr/awesome-web-scraping) list.

## Compliance note

Google Maps data is public business info — but respect GDPR and outreach laws (CAN-SPAM, opt-outs). Scrape public data, don't republish personal info, and honor platform ToS.

---

*Questions? Drop them in the comments — happy to share run inputs for specific niches.*

---

## Resources

- [Apify](https://console.apify.com/sign-up?fpr=dx06p) — the scraping platform used in this tutorial (free credits on signup)
- [g4dr/apify-skills](https://github.com/g4dr/apify-skills) — Agent Skills that run these pipelines from a prompt
- [g4dr/awesome-web-scraping](https://github.com/g4dr/awesome-web-scraping) — curated list of scraping tools, proxies and actors
