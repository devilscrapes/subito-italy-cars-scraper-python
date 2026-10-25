<p align="center"><a href="https://apify.com/DevilScrapes/subito-italy-cars"><img src="assets/banner.svg" alt="Subito Italy Car Scraper — Python example" width="100%"></a></p>

# Subito Italy Cars Scraper in Python: subito.it Auto searches to pandas, JSON and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/subito-italy-cars"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/subito-italy-cars-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/subito-italy-cars-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

What you have here is a compact Python wrapper, with tests, for the **[Subito Italy Car Scraper](https://apify.com/DevilScrapes/subito-italy-cars)**, which runs on Apify. You give it a subito.it Auto search, say every Fiat Panda advertised in Lombardy, and it hands back a table in which each row is one car: asking price in euros, kilometres on the clock, registration year, power in kW and CV, fuel, gearbox, body style and the town, province and region where the car stands. The fetching happens on Apify's servers, so your laptop only runs a small script and receives `results.json` and `results.csv`. From there it is ordinary pandas work: depreciation curves, regional price gaps, mileage-adjusted comparisons.

> **Actor:** [Subito Italy Car Scraper](https://apify.com/DevilScrapes/subito-italy-cars) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [subito-italy-cars-scraper-javascript](https://github.com/devilscrapes/subito-italy-cars-scraper-javascript)

## 🔥 What you get

- **Aim with a subito link, not a form**: subito encodes filters in the path, so `/annunci-lombardia/vendita/auto/fiat/panda/` already means Fiat Panda in Lombardy. Paste such a link into `searchUrl`, or leave it blank for the nationwide used-car feed.
- **Numbers stay numbers**: `price` is whole euros, `mileage_km` is an integer odometer reading and power arrives twice, as `engine_power_kw` and `engine_power_hp` (the Italian CV figure).
- **Make, model and trim split apart**: `make`, `model` and `version` come from subito's own car taxonomy rather than from guessing at the seller's headline.
- **Italian geography in three tiers**: `region` (regione), `province` (provincia) and `location` (the comune) are separate columns, ready for grouping.
- **Full ad text without a second request**: the Italian `description` and the photo links come from the results pages themselves; `enrichDetails` only swaps in the image list from each ad's own page.
- **No duplicate rows across pages**: an ad seen on an earlier page is not written or charged again.
- **Residential exits enforced**: the Actor always adds the RESIDENTIAL proxy group and stops with a clear message when the account cannot use it, instead of quietly returning an empty dataset.

## 🎯 Use cases

### Draw a depreciation curve for one model

Collect a few hundred ads for a single make and model, group by `year` and plot the median `price`. The slope tells you how fast that car loses value in Italy, and the spread inside each year shows how much condition and kilometres matter.

### Compare prices between regions

Run the same make and model path for two or three regions, for example Lombardia, Lazio and Campania, then concatenate the CSV files. A pivot on `region` and `year` shows whether the same car is cheaper in the south, which is the question every cross-region buyer asks.

### Keep a nightly price log

Point cron at `python main.py --out data/$(date +%F)` and you collect one snapshot per day. Merging snapshots on `listing_id` reveals which ads were cut in price, which sold or were withdrawn, and how long a typical car stays online.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/subito-italy-cars-scraper-python.git
cd subito-italy-cars-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Subito Italy Car Scraper](https://apify.com/DevilScrapes/subito-italy-cars), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without the CLI scaffolding, a run is the snippet below. `call()` waits for the Actor to finish, then `iterate_items()` streams the dataset in pages so a 1,000-car pull does not need to fit in one response.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/subito-italy-cars").call(run_input={'searchUrl': 'https://www.subito.it/annunci-lombardia/vendita/auto/fiat/panda/', 'maxResults': 10, 'enrichDetails': False, 'proxyConfiguration': {'useApifyProxy': True, 'apifyProxyGroups': ['RESIDENTIAL']}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` passes `input.json` through untouched. The sample asks for Fiat Panda ads in Lombardy, keeps detail enrichment off and stops at 10 cars, so a test costs about $0.22 (the start fee plus ten rows). To change the target, open subito.it, set make, model, region and any price or year filters, then copy the address bar into `searchUrl`. `maxResults` goes up to 1,000.

```json
{
  "searchUrl": "https://www.subito.it/annunci-lombardia/vendita/auto/fiat/panda/",
  "maxResults": 10,
  "enrichDetails": false,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ]
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchUrl` | string | no | `""` | — |
| `maxResults` | integer | no | `50` | — |
| `enrichDetails` | boolean | no | `false` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}` | — |

## 📤 Output

Each row is one car ad. The ID, link, headline and scrape time are always there; every other field is null when the seller skipped it. `posted_date` is subito's own local timestamp in `YYYY-MM-DD HH:MM:SS` form, not ISO and not UTC, while `scraped_at` is a proper UTC ISO string. `seller_name` can hold an advertiser's name, so treat it as personal data whenever `seller_type` is `private`.

| Field | Type | Always present |
|---|---|---|
| `listing_id` | string | yes |
| `listing_url` | string | yes |
| `title` | string | yes |
| `make` | string or null | no |
| `model` | string or null | no |
| `version` | string or null | no |
| `year` | integer or null | no |
| `price` | integer or null | no |
| `currency` | string or null | no |
| `mileage_km` | integer or null | no |
| `fuel_type` | string or null | no |
| `transmission` | string or null | no |
| `engine_power_hp` | integer or null | no |
| `engine_power_kw` | integer or null | no |
| `body_type` | string or null | no |
| `color` | string or null | no |
| `first_registration` | string or null | no |
| `location` | string or null | no |
| `region` | string or null | no |
| `province` | string or null | no |
| `seller_type` | string or null | no |
| `seller_name` | string or null | no |
| `photo_urls` | array | no |
| `description` | string or null | no |
| `posted_date` | string or null | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| result-row | $0.002 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/subito-italy-cars) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

A first analysis with pandas (`pip install pandas`): per registration year, count the ads and take the median price and median kilometres, then add how many of those kilometres each car covered per year of age. The summary prints to the terminal and is saved as `by_year.csv`.

```python
import json

import pandas as pd

cars = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
cars = cars.dropna(subset=["price", "year"])
cars["year"] = cars["year"].astype(int)

this_year = pd.Timestamp.now().year
age = (this_year - cars["year"]).clip(lower=1)
cars["km_per_year"] = cars["mileage_km"] / age

by_year = (
    cars.groupby("year")
    .agg(
        ads=("listing_id", "count"),
        median_eur=("price", "median"),
        median_km=("mileage_km", "median"),
        km_per_year=("km_per_year", "median"),
    )
    .sort_index(ascending=False)
)
by_year.to_csv("by_year.csv")
print(by_year.round(0).to_string())
```

## ❓ FAQ

### How do I target one make, model and region?

Build the search on subito.it and copy the URL. The site writes those filters into the path, as in `annunci-lombardia/vendita/auto/fiat/panda/`, and query parameters such as a price range survive as well. The Actor adds its own page number and walks forward from there.

### Why did my run fail instead of returning zero rows?

When the very first results page yields no ads, the Actor raises an error rather than reporting success with an empty table. Usually the URL points at a search with no matches, or is not an Auto search at all. Open it in a browser and check that cars appear.

### Can I run it on the free Apify plan?

Only if your account can use the RESIDENTIAL proxy group. The Actor insists on it for subito.it and fails at start-up with an explicit message when it is missing, so you find out in seconds rather than after a long empty run.

### Is `enrichDetails` worth switching on?

Rarely for analysis. Prices, specs, location and the full description already come from the results pages. Enrichment opens every ad a second time just to replace `photo_urls` with the image list from the ad page, so it roughly doubles the requests.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/subito-italy-cars) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Subito Italy Car Scraper on Apify Store](https://apify.com/DevilScrapes/subito-italy-cars)
- [JavaScript version of this example](https://github.com/devilscrapes/subito-italy-cars-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AutoScout24 Germany Car Scraper](https://apify.com/DevilScrapes/autoscout24-germany-cars) — the German used-car market, handy for cross-border price gaps.
- [coches.net Spain Car Scraper](https://apify.com/DevilScrapes/coches-spain-cars) — Spanish car ads from coches.net, another southern European benchmark.
- [Njuškalo Car Scraper (Croatia)](https://apify.com/DevilScrapes/njuskalo-croatia-cars) — Croatian vehicle classifieds for the Adriatic side of the comparison.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/subito-italy-cars) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Subito.it.
