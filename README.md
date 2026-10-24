# amazon price scraper: Structured Buy Box data in one request, with real credit math and a daily monitoring setup

Scraping Amazon prices is easy right up until it isn't. The Python part is trivial; the part that kills most projects is the CAPTCHA wall, the rotating proxy pool you have to maintain, and the brittle HTML parser that breaks every time Amazon tweaks a class name. **ScraperAPI** takes that whole layer off your plate: you send a request with an ASIN, it routes through its proxy infrastructure, and it returns structured JSON with the Buy Box price, list price, seller, and rating already parsed.

Plans start at **$49/month** (100,000 credits), there's a free tier with 1,000 credits, and a 7-day trial with 5,000 credits that doesn't ask for a card. This article covers all seven paid plans, the actual per-page cost of Amazon scraping (which is where most reviews get vague), what the Amazon endpoints return, and a working setup for daily price and seller monitoring.

> **Quick pick:** for most individual users and small teams running a price tracker, the **Hobby plan at $49/month ($44.10 on annual billing)** covers it — 100,000 credits works out to roughly 20,000 Amazon product pages per month. If you're not sure yet, register for free and you get 1,000 credits with no card required. 👉 [See the full plan list and current prices](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What an amazon price scraper actually has to survive

A reliable price scraper needs four things handled at once: IP rotation so Amazon doesn't flag you, CAPTCHA solving, JavaScript rendering for pages that need it, and parsing that doesn't collapse when the page markup changes.

ScraperAPI's approach is a single HTTP endpoint. You pass your API key and the target URL (or an ASIN against one of its structured endpoints), and the service handles proxies, retries, CAPTCHAs, and rendering on its side. A third-party review from 2026 describes the same picture: rotating proxy pools with premium residential and mobile IPs, automatic retries, sticky sessions that last 15 minutes, and unlimited bandwidth — you pay per request, not per GB.

The part that matters most for price tracking is the **Structured Data endpoints**. Instead of fetching raw HTML and writing your own parser, you hit a dedicated Amazon endpoint and get clean JSON back. The same endpoint family covers Walmart, eBay, Home Depot, and Google Shopping, so if your tracking expands past Amazon later, you're not rewiring everything.

## The full plan lineup: seven paid tiers plus a free plan

Here's every plan currently listed on ScraperAPI's pricing page. All plans include JavaScript rendering, premium residential and mobile IPs, CAPTCHA and anti-bot bypassing, JSON auto-parsing, DataPipeline, crawler access, and a 99.9% uptime guarantee. Annual billing cuts every price by 10%.

| Plan | Credits/mo | Concurrent requests | Monthly price | Annual (per month) | Link |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0 | $0 | [ Start free](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | $49 | $44.10 | [ Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | $149 | $134.10 | [ Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | $299 | $269.10 | [ Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | $475 | $427.50 | [ Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | $975 | $877.50 | [ Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | $1,975 | $1,777.50 | [ Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Custom quote | Custom quote | [ Contact for a quote](https://www.scraperapi.com/pricing?fp_ref=coupons) |

A few details that only show up if you read the fine print, and that genuinely affect which plan you should pick:

- **Geotargeting**: Hobby and Startup are limited to US and EU locations. Country-level targeting for other Amazon marketplaces requires Business ($299) or above. If you're tracking prices on Amazon Japan or Amazon India specifically by country, factor that in.
- **Credits don't roll over**. Unused credits expire at the end of the billing cycle.
- **Overage options differ by tier**: auto-upgrade (opt-in) is available from Hobby through Professional, and pay-as-you-go beyond your limit is only available from Scaling up. On the lower tiers, scraping simply pauses until your credits renew.
- **Analytics retention**: the lower plans keep dashboard analytics for 30 days; Business and above keep them indefinitely.

That last trio matters more than it sounds. A review of the service notes that "credits don't roll over, and overage exists only from Scaling up" is a fair complaint — you can change plans at any time, but if your ASIN count spikes mid-month on Hobby, you either wait for the reset or upgrade.

## The real cost: an Amazon page costs 5 credits, not 1

This is the number most comparison articles skip, and it changes your plan math completely.

ScraperAPI charges in credits, and the cost per request depends on the target site:

- Standard pages: 1 credit
- **Amazon, Walmart, eBay: 5 credits**
- Google and Bing (including subdomains): 25 credits
- LinkedIn: 30 credits

So a "100,000 credits" Hobby plan is 100,000 requests for plain pages, but **20,000 Amazon product pages**. At $49/month that's about $2.45 per 1,000 Amazon pages, dropping to roughly $2.21 on annual billing. The flip side: Amazon is one of the cheapest major targets on the platform — five times cheaper than scraping Google SERPs.

Add-on parameters can inflate that quickly if you're not careful:

- `render=true` (JavaScript rendering): +10 credits
- `premium=true` (premium proxy pool): +10 credits
- `ultra_premium=true`: +30 credits (paid plans only)
- Bypassing Cloudflare, DataDome, or PerimeterX: +10 credits each when triggered

For most Amazon price work you won't need any of these — the structured Amazon endpoints already handle the heavy lifting, and parameters like `country_code`, `output_format`, and `session_number` cost nothing extra.

Two tools keep you honest. Before running anything at scale, you can query the exact cost of a URL through the `account/urlcost` endpoint or the Domain Cost Estimator in the dashboard, and you can set a `max_cost` per request so a misconfigured job can't silently burn your balance. Every response also includes a `sa-credit-cost` header showing exactly what that request consumed.

Billing has one more wrinkle worth knowing: you're charged for successful requests — status codes 200 and 404 — and for requests you cancel yourself before the 70-second mark. Failed requests blocked by Amazon don't bill. That's a meaningfully different model from tools that charge per attempt.

## What the Amazon endpoints actually return

For a price scraper specifically, three structured endpoints do most of the work.

**Product endpoint.** One request per ASIN returns eleven core fields without any login: title, ASIN, Buy Box price, list price (the struck-through original), currency, star rating, review count, star distribution, Buy Box seller, availability, and the first few featured reviews. Deeper parsing can pull 50+ additional fields if you need them.

**Offer endpoint.** This one returns the `listings` array — every seller offer on the page, each with `seller_name`, `price`, `shipping_price`, `condition`, `pinned_offer`, and whether it's fulfilled by Amazon. For brand owners, this is the endpoint that turns a price scraper into an enforcement tool.

**Search endpoint.** Returns structured data from Amazon search results, useful for category monitoring and discovering competitor ASINs you're not tracking yet.

Marketplace switching is built in: `country_code=gb` targets the UK, `de` Germany, and a `domain` parameter handles `co.uk`, `com.au`, and similar. Output comes as JSON by default, CSV if you ask, and a `condition` filter lets you separate new from used offers.

The official documentation walks through a complete daily monitoring workflow that's worth copying:

1. Build an Offer request in the API Playground for a test ASIN (their example uses `B0BFC7WQ6R`), check the cost preview before running it.
2. Prepare two reference files: an authorized seller list, and a CSV mapping each ASIN to its minimum advertised price (MAP).
3. Switch to **Async** mode and submit your whole catalog as a batch — one ASIN endpoint URL per line.
4. Poll progress via the `statusUrl` from the batch response, or skip polling entirely by configuring a destination in the dashboard and adding a `callback`, so finished results land in your storage automatically.
5. Schedule the whole batch with a cron job — the docs' example runs it daily at 6am.

The result is a daily automated check that flags unauthorized sellers and prices below your MAP threshold across the entire catalog. There's also a lower-code path: DataPipeline accepts up to 100,000 URLs, keywords, or ASINs per project, with built-in scheduling and webhook delivery, so you don't even need the cron job.

If you want to try this workflow before committing to a plan, the free credits are enough to test a handful of ASINs end to end. 👉 [Sign up for the free trial and test your first ASIN](https://www.scraperapi.com/pricing?fp_ref=coupons)

## How it compares with other Amazon price scrapers

The "amazon price scraper" search results are crowded, so here's the honest landscape based on verified public info — no links, just facts to compare against.

| Tool | Pricing model | Notable difference |
| --- | --- | --- |
| ScraperAPI | Credit-based subscription, 7 tiers, Amazon at 5 credits/req | Structured Amazon endpoints + async batches + DataPipeline scheduling |
| Apify | Pay-per-event, no monthly subscription | Ready-made Amazon actor covering prices, reviews, Prime status across 11 marketplaces |
| Bright Data | Claims 5,000 records/month free on its Amazon scraper API | Time-stamped price snapshots for movement monitoring (its free-tier page dates from 2023, so verify current terms) |
| ShoppingScraper | API + dashboard | Stores up to 12 months of price history for any product you track |
| AmazonScraperAPI | From $0.90 per 1,000 successful requests ($0.50 on custom plans) | 1,000 free requests on signup; async batches accept up to 1,000 ASINs per POST |

Two things worth weighing honestly. First, if you want **zero code**, Apify's pay-per-event model is genuinely easier to start with than any API — you're renting a finished scraper instead of building one. Second, if you want **built-in historical price data**, ShoppingScraper stores 12 months of it for tracked products; with ScraperAPI you receive the current snapshot and store the history yourself in your own database or spreadsheet. That's extra work, but it also means you own the data and can combine it with your own margin and repricing logic.

Where ScraperAPI earns its keep is the middle ground: you have some code, you want predictable pricing per successful request, and you want the daily monitoring job (offers, MAP violations, multi-marketplace coverage) to run without babysitting proxies.

## Free credits, trial, and what happens if it doesn't work out

The entry cost is zero, in two stages. Signing up gets you **1,000 free API credits** with up to 5 concurrent connections — enough to pull around 200 Amazon product pages and confirm the data structure works for your use case. New accounts also get a **7-day trial with 5,000 credits**, and neither step requires a credit card.

If you subscribe and change your mind, the published policy is a **7-day no-questions-asked refund** — contact support through the dashboard and they process it. You can also cancel anytime from the billing page with no cancellation fee.

That combination makes the decision reversible at every stage: test with free credits, pay, and still have a week to walk away with your money. 👉 [Check current prices and start the 7-day trial](https://www.scraperapi.com/pricing?fp_ref=coupons)

## FAQ

**How many Amazon products can each plan track per day?**
Divide the monthly credits by 5. Hobby (100,000 credits) covers about 20,000 product page pulls a month — roughly 660 ASINs checked daily. Startup (1,000,000 credits) supports about 6,600 daily checks, which is enough for a full mid-size catalog plus competitor watch.

**What does it cost to scrape 1,000 ASINs every day?**
At 5 credits per request, 1,000 daily ASINs means 30,000 requests a month, or 150,000 credits — that overshoots Hobby's 100,000, so the Startup plan at $149/month (annual $134.10) is the realistic floor. Buy Box monitoring via the Offer endpoint bills the same 5-credit rate per page.

**Does ScraperAPI store my price history for me?**
No published history-storage feature appears in its documentation — the endpoints return current snapshots, and keeping historical price series is on you. If turnkey history is essential, that's the one area where a specialist like ShoppingScraper (12 months of stored history) does something ScraperAPI doesn't.

**Am I charged when Amazon blocks a request?**
No. Billing applies to successful responses (status 200 and 404) and to requests you cancel within the first 70 seconds. The docs recommend a 70-second client timeout, since heavily protected pages can take up to 60 seconds.

## Before you pick a plan

Match the plan to the job rather than to the biggest discount:

- Testing an idea or tracking a few dozen ASINs → the free 1,000 credits and the 7-day 5,000-credit trial will carry you; don't pay yet.
- A personal price tracker or small seller account → **Hobby at $44.10/month annual**; 20,000 Amazon pulls a month is a lot of price checks.
- Full-catalog daily monitoring, or you need sellers and MAP checks alongside prices → Startup, and add the Offer endpoint to your batch.
- Country targeting outside the US/EU matters → start at Business, since Hobby and Startup are capped to US and EU geotargeting.

One last check before subscribing: run your real ASIN list through the Domain Cost Estimator or the `urlcost` endpoint so you know your true credit multiplier, not an assumed one. Then take the annual billing on whichever tier fits — the 10% saving is applied automatically. 👉 [Open ScraperAPI's pricing page and pick your plan](https://www.scraperapi.com/pricing?fp_ref=coupons)
