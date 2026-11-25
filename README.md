# Scrape Google Search Results: Why TLS Fingerprints and AI Overviews Break DIY Scripts, What to Demand From an API, and How Far $49 Gets You

Two things now decide whether a Google scraper survives past its first week. One is whether your TLS handshake looks like Chrome's, because fingerprinting has become a primary detection signal and a perfectly configured Playwright instance still gets flagged when the handshake doesn't match. The other is whether your parser knows that the AI Overview block sitting above the organic results is built from an entirely different DOM than the ten links below it.

ScraperAPI's Google Search endpoint covers both for 25 credits per request, which puts the $49 Hobby plan at roughly 4,000 Google searches a month. What follows is the pagination math, the SERP blocks worth collecting, the field list to check before paying any provider, and where a self-built script stops being the cheaper option.

> **Most people should start on Hobby.** $49 a month buys 100,000 credits, a Google request costs 25, so you get about 4,000 SERPs, or roughly 130 a day. That covers daily rank tracking on a few hundred keywords. The free tier's 1,000 credits sound generous until you divide by 25 and get 40 Google searches, which is enough to inspect the JSON shape and nothing more.
> 👉 [See ScraperAPI's current plans and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## What each plan actually buys, in Google searches

ScraperAPI prices by domain, not by plan tier. A standard page costs 1 credit, Amazon 5, Google and Bing including their subdomains 25, LinkedIn 30, and sites sitting behind Cloudflare, DataDome or PerimeterX add 10 credits when ScraperAPI runs its bypass logic. None of those multipliers change when you upgrade. A larger plan buys a larger allowance and usually a better price per credit, not cheaper requests.

That makes the arithmetic short. Divide the monthly credit allowance by 25.

| Plan | Monthly credits | Google SERPs / month | Price | Best for | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | ~40 | $0 | Inspecting the response format before you commit | [Create a free account](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | ~4,000 | $49/mo, about $44/mo billed annually | Daily rank tracking on a few hundred keywords | [Start with Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | ~200,000 | Listed on the pricing page | Multi-country tracking, agency-scale collection | [Check the Scaling rate](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | Custom quote | Committed volume and concurrency terms | [See Enterprise options](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

ScraperAPI also sells Startup, Business, Professional and Advanced tiers between Hobby and Enterprise. Because the per-request multiplier is fixed per domain, you can size any of them yourself: credits divided by 25 gives Google capacity, and the dashboard's Domain Cost Estimator is the authoritative check for any specific URL.

Three billing details matter more than the headline prices. Credits reset at each renewal and unused credits are lost, so an oversized annual plan wastes money quietly. Annual billing takes exactly 10% off, which puts Hobby near $44 a month. Running past your allowance on Hobby, Startup or Business gets you a 429 rather than a bill, because pay-as-you-go is a Scaling-and-above feature. Auto-upgrade, which moves you to the next tier when credits run dry, is opt-in and available on Hobby through Professional.

At Hobby pricing, 4,000 Google searches works out to about $12.25 per 1,000 SERPs, or $11 per 1,000 on annual billing. Keep that number in mind when you compare any provider's per-search rate.

## What's on a Google SERP, and which blocks are worth the request

A SERP is whatever Google returns for a query, and that page is now a stack of modules rather than a list. Featured snippets, paid ads, video carousels, People Also Ask boxes, the local map pack, knowledge panels, image and shopping blocks, related searches at the bottom. Google is one of the richest public data sources online precisely because every query returns this much structured information.

The organic results are the part everyone collects. The two blocks people skip are worth more for keyword and competitor work. People Also Ask is an accordion: the question list often renders without interaction, but the answer text needs a click to expand. Related Searches at the page footer maps the query space around your keyword and shows you what Google thinks is adjacent. Both need their own locators and, for PAA answers, an interaction step.

Then there's what now sits above all of it. AI Overviews appear at the top of a large share of informational queries, which pushes organic results further down the page. The summary block uses a completely different DOM structure than traditional results, so a scraper targeting the standard result container will miss the entire section without ever erroring out. Extracting it means locating a separate container and handling dynamic expand and collapse behavior.

## Doing it yourself: the URL, the offsets, and the selectors that will break

The request format is not the hard part. A search URL is `https://www.google.com/search?hl=en&q={query}`, and pagination is an offset: add `&start=10*(page-1)` from page two onward, since a raw Google page returns about ten organic results. Set `hl` for interface language, `gl` for country, add `num` if you're experimenting with larger pages.

Parsing is where tutorials look tidy. A Playwright script waits for `div.N54PNb`, and inside each container pulls the title from `h3.LC20lb`, the href from `a.zReHs`, the snippet from `div.VwiC3b`, and derives the domain by splitting the URL on slashes. In a code sample this works.

In production it holds until Google changes a class name. Then you get zero results with a 200 status code and no exception, which is worse than a block because nothing alerts you. AI Overviews made this structural rather than incidental: every layout change now adds or moves a block, and each one is a separate parser to maintain. A 2026 SERP with AI summaries and rich snippets demands more parsing robustness than the same page did two years ago, and rich results that aggregate Wikipedia or IMDb data have their own stable-looking structures that still move.

Detection sits on a separate layer. Random delays, rotating user agents and a residential pool are the common recipe. On top of that, TLS fingerprinting means the handshake your HTTP client presents has to resemble a real browser's; a stock Playwright Chromium build doesn't, and patched builds that mimic Chrome's fingerprint are now treated as practically necessary at any meaningful scale. Add 403 handling with exponential backoff, because immediate retries on a rate limit just extend it.

None of this is impossible. It's a maintenance job with an on-call component, and it's the actual dividing line between building and buying.

## The field checklist to run before you pay any SERP provider

Most tutorials show you a request and stop. The response schema is what you should be comparing, because switching providers later means rewriting your downstream pipeline. A response worth paying for gives you:

- `organicResults` with title, url, displayedUrl, description and an explicit `position` per result. Without position you're inferring rank from array order, which breaks the moment a block is inserted.
- `resultsTotal`, parsed from the "About X results" line. Cheap to include, useful for volume signals on long-tail queries.
- `peopleAlsoAsk` with question text, and the answer and URL when available.
- `relatedQueries` as an array of title and URL pairs.
- `paidResults` and `paidProducts`, the latter with prices, so you can see who is buying the query and at what price point.
- `aiOverview` with summary text and its reference list, where the provider supports it.
- `htmlSnapshotUrl`, so you can audit a bad parse against the raw page instead of guessing.
- The query echo: `gl`, `hl`, `uule` for exact location, device, page number, results per page, and a `hasNextPage` flag.

Two more capabilities separate providers in practice. One is results per page: managed services let you set 10, 50 or up to 100 per request, which cuts ten paginated requests down to one and changes your cost per keyword. The other is filtering at request time, including exact match, site and related-site scoping, words in title or URL, file type and date ranges, so you don't pay to download pages you'll discard.

ScraperAPI's Google Search endpoint returns structured JSON rather than raw HTML, and its own walkthrough pulls organic results, People Also Ask questions and YouTube video results from a single query before writing them to CSV. For teams wiring this into an agent rather than a script, there's an MCP server that runs the same scrape from Claude Code, Cursor or any MCP client. If AI Overview text is a hard requirement for your project, confirm coverage with support before you buy rather than assuming it, since it isn't part of every structured endpoint.

👉 [Check what the Google Search endpoint returns](https://www.scraperapi.com/solutions/structured-data/google-search-scraper/?fp_ref=coupons)

## Localized results depend on your IP, not your locale

Setting `locale` and `timezone_id` in Playwright simulates a location. It doesn't change where the request comes from, and IP address location carries more weight than browser locale settings in deciding which regional SERP Google serves. A "German" scrape from a US datacenter IP returns a US SERP with German formatting, which is a subtle way to publish wrong rank data for months.

Reliable per-country results require exit IPs in that country: a residential pool you manage yourself, or a provider that sells geo-targeting as a request parameter and swaps the exit node server-side. Oxylabs, for example, takes a `geo_location` value and a `locale` in the request payload and returns results as if you were physically in that country.

Before choosing a plan, list the countries you actually need and check coverage. If you're tracking ten markets, the question isn't the credit price, it's whether all ten are supported at the geo granularity you need, down to city level via UULE parameters where local SEO is the point.

## Matching volume to a plan

Work backward from queries times frequency times markets.

A few hundred keyword spot checks a month: 300 keywords is 7,500 credits, comfortably inside Hobby, and the free tier's 40 searches won't get you through a single audit. Daily rank tracking on 500 keywords in one country: 15,000 SERPs a month at 25 credits each is 375,000 credits, which puts you into the tier above Hobby. The same 500 keywords across five countries is 75,000 SERPs, or 1.875 million credits, and that's Scaling territory at 200,000 Google searches.

Compare that to the DIY bill. Residential proxy bandwidth billed per gigabyte, a patched browser build to maintain, selector updates every time a block moves, retry logic for 403s, and monitoring to catch the silent zero-result failure. Those costs don't appear on an invoice, which is why self-built scrapers look cheaper right up until the week someone has to fix them.

## FAQ

**Is scraping Google results legal?**
Oxylabs states in its own tutorial that scraping publicly available data, including Google SERP data, is legal, while noting that this varies from one situation to another and recommending legal advice for your specific case. That's a vendor position, not a legal conclusion. Google's Terms of Service and data protection law such as GDPR or CCPA apply independently, and if you're collecting at scale for a commercial product, the ToS question is the one to put in front of a lawyer.

**Can I get the AI Overview text?**
Some providers expose it as a structured field, sometimes behind a separate engine or an AI Mode flag. It requires locating a container that isn't the organic result container and handling expand and collapse, so verify it explicitly with whichever provider you choose.

**Do I have to click People Also Ask to get the questions?**
The question list usually renders on load. The answer text typically needs an expansion click, which means browser automation or a provider that has already done the interaction for you.

**What do I do about 403s?**
Detect the status code and retry with exponential backoff rather than immediately, and rotate the exit IP. Repeated instant retries from the same IP confirm to Google that you're a bot.

**How many results can I get per request?**
On raw Google HTML, about ten organic results per page, paginated with `start=10*(page-1)`. Managed APIs let you set results per page anywhere from 1 to 100, which is the single biggest lever on cost per keyword.

**What happens when I hit the concurrency limit?**
ScraperAPI returns a 429 asking you to bring concurrency back within your plan's limit. There's no extra charge; the request is turned away rather than queued.

## Which plan fits

Running a one-off keyword audit or building a prototype: start on Free to see the JSON, then Hobby at $49 once you're past 40 searches, which will be within the first hour.

Tracking a few hundred keywords daily in one or two countries: Hobby, and switch to annual billing for the 10% if you already know you'll keep it.

Tracking thousands of keywords across multiple markets, or feeding SERP data into a product: the tier above Hobby, sized by credits divided by 25, with pay-as-you-go once you reach Scaling so a spike doesn't stall the pipeline.

Two things to confirm before you enter a card number. First, whether your plan's allowance is measured in credits with the Google multiplier applied, because 100,000 credits and 100,000 searches are very different numbers, and the Domain Cost Estimator in the dashboard settles it per URL. Second, whether geo-targeting covers every country you're reporting on, since a plan that's cheap per search but can't return the right regional SERP is the most expensive option on the list.

👉 [Start with the free tier, then size your plan by credits per Google search](https://www.scraperapi.com/pricing/?fp_ref=coupons)
