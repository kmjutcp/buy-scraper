# Best Buy Scraper: What Can You Actually Extract, Why Do the Selectors Keep Changing, and What Does Each Request Cost? (SKU vs BSIN, the Review Endpoint, and Three Scraping Routes Compared)

Best Buy publishes prices, stock status, spec sheets and review counts across thousands of SKUs, and all of it sits behind bot protection that assumes you are not a browser. A working Best Buy scraper needs three things settled before parsing starts: a US residential IP, a way past the bot check, and rendering for pages that load products as you scroll. After that, the data is more accessible than the page source suggests. Product pages carry an embedded JSON payload with specifications, buying options and FAQs, reviews come from their own JSON endpoint, and search results are driven by three URL parameters.

The route you choose decides both your maintenance load and your bill. A plain HTML request, a rendered request and a bot-protected request are priced differently, and Best Buy pushes you toward the expensive end of that list. What follows is the field list worth pulling, the identifiers people mix up, the three ways teams get this data today, and how to size the spend before you commit.

> Most readers building a Best Buy price feed should start small, run a handful of real Best Buy URLs through the Domain Cost Estimator in the dashboard, and only then size a plan. A new account comes with 1,000 API credits and a maximum of 5 concurrent connections, which is enough to measure what a rendered Best Buy page costs you before paying for volume. 👉 [Check current plans and credit allowances on ScraperAPI's pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans at a glance

ScraperAPI runs a free tier plus seven paid tiers: Hobby, Startup, Business, Scaling, Professional, Advanced and Enterprise. The differences that matter for a Best Buy crawl are how many credits each tier allows, how the account behaves when credits run out, and how much rendered concurrency you can push.

| Plan | Credit allowance | Overage and renewal behavior | Fits |
| --- | --- | --- | --- |
| Free | 1,000 credits on signup, max 5 concurrent connections | Credits renew on the billing cycle | Timing a handful of Best Buy URLs before you buy |
| Hobby | Scales with the tier | Auto-upgrade available when credits run out | First production feed on a small SKU list |
| Startup | Scales with the tier | Auto-upgrade available | Category-level monitoring |
| Business | Scales with the tier | Auto-upgrade available | Multi-category price tracking |
| Scaling | Scales with the tier | Auto-upgrade plus pay-as-you-go up to a monthly cap | Deal windows with unpredictable volume |
| Professional | Scales with the tier | Auto-upgrade plus pay-as-you-go | Larger catalogs, steady hourly runs |
| Advanced | Scales with the tier | Pay-as-you-go up to a monthly cap | High-volume refreshes |
| Enterprise | Negotiated | Pay-as-you-go; higher rendered concurrency on request | Continuous crawling |

Per-plan prices differ between monthly and annual billing, and annual plans can renew early for a fresh batch of credits. Those numbers move, so check them on the pricing page rather than trusting a table in an article. 👉 [See every plan's current price and credit allowance](https://www.scraperapi.com/pricing?fp_ref=coupons)

One structural detail is worth more than the plan names. ScraperAPI prices per request by how hard the target is. A standard page costs 1 credit. Amazon costs 5, Google and Bing cost 25, LinkedIn costs 30. Sites behind bot protection such as Cloudflare, Datadome or PerimeterX add 10 credits per request when ScraperAPI bypasses them. JS rendering costs 10 credits on its own, 25 with premium proxies, and 75 with ultra premium proxies. For a retailer like Best Buy, those multipliers matter more than the size of the plan you pick, and the Domain Cost Estimator in the dashboard will give you the exact figure for any URL.

## The fields worth pulling, and the four identifiers people confuse

A full Best Buy product record runs to 29 fields. The ones that earn their place in a price-monitoring or analytics pipeline:

| Field | Type | What it gives you |
| --- | --- | --- |
| `skuId` | string | Best Buy's own 7-digit SKU |
| `bsin` | string | 10-character canonical product ID that groups color and size variants under one product page |
| `upc` | string | Universal Product Code from the barcode |
| `modelNumber` | string | Manufacturer model number |
| `currentPrice` / `regularPrice` | number | Customer-facing price and non-sale list price |
| `totalSavings` / `totalSavingsPercent` | number | Absolute and percentage discount |
| `onSale` | boolean | True when current price is below regular price |
| `availability` | string | Best Buy's `dotComDisplayStatus`, e.g. active, archived, coming-soon |
| `averageRating` / `reviewCount` | number / integer | Star average (1.0 to 5.0) and total reviews |
| `categoryPath` / `primaryCategoryId` | array / string | Breadcrumb from root to leaf, plus Best Buy's category ID |
| Specifications block | key-value pairs | Spec table, highlights and features |

The identifier question trips up more projects than any parsing bug. A 7-digit SKU identifies one sellable item at Best Buy and nothing anywhere else. The 10-character BSIN is what you group on when you want one row per product rather than one row per color. Neither travels.

For cross-retailer matching, `modelNumber` and `upc` are the reliable join keys. A SKU will not match at any other retailer, so a comparison table built on SKU alone silently produces zero matches. Build the join on model number first and UPC as a second pass, then keep the SKU as the local key.

Specifications are the other trap. Field sets vary heavily by category, and a laptop spec block shares little with an appliance one. Do not assume a field that exists on laptops exists on refrigerators, and do not design a schema against one category before sampling several.

## Three routes into the same catalog, with different failure modes

Published scrapers for Best Buy use at least three distinct approaches. They are not variations on a theme; they fail for different reasons and cost different amounts.

| Route | How it works | Strength | Where it breaks |
| --- | --- | --- | --- |
| Embedded JSON on the product page | Locate the script tag containing `productBySkuId`, extract the JSON after `.push(`, then read `events` where `type == "next"` for the product payload | One request returns product info, features, buying options, FAQs and spec categories together | Breaks whenever the page stops shipping that payload shape, and needs bracket matching rather than a JSON parser on the raw script |
| Direct API read | Query the structured endpoint behind the page and take specifications as key-value pairs | No HTML table parsing, and the response is already typed | Depends on the endpoint staying reachable and its schema staying stable |
| Rendered DOM plus CSS selectors | Load the page in a browser, scroll, then read elements | Works on listing pages where you want what a shopper sees | Selectors drift, and listing pages appear to serve more than one DOM version |

That last row deserves evidence. Two published Best Buy scrapers, written against the same retailer, share no selectors for the same elements. One reads product cards from `li.product-list-item` and the price from `div.customer-price.medium`. The other reads `.product-grid-view-container li` and `[data-testid="price-block-customer-price"]`. For the primary image one uses `div.product-image.m-100 img`, the other `img[data-testid='product-image']::attr(srcset)`. Both were presumably correct when written. Treat a selector-only pipeline as something that needs monitoring and a fallback, not as a fixed asset.

A reasonable rule: if you need specifications and FAQ content, take the embedded JSON or the API route and skip rendering where you can. If you need to mirror what a shopper sees on a category page, rendering is unavoidable. If you are building both, keep them as separate jobs so a DOM change does not take down your pricing feed.

Worth knowing on the API question generally: official product APIs tend to expose core fields but miss presentation-layer detail such as promo badges, special pricing and store-specific phrasing. If your use case depends on how a deal is displayed rather than the numbers alone, plan to read the page.

## Reviews come from their own endpoint, not the page

Parsing review HTML is wasted effort at Best Buy. Reviews are served from a JSON endpoint that takes the SKU directly:

`/ugc/v2/reviews?page=1&pageSize=20&sku={skuId}&sort=MOST_RECENT`

The response carries `totalPages` and `topics`, so you can page through without guessing when to stop. Pull page one, read the page count, then fan the rest out concurrently. Twenty reviews per request keeps the payload small and the page count predictable.

Ratings on listing pages can be lifted from the accessibility text with a regex over the pattern `Rating\s+([\d.]+)\s+out of\s+\d+\s+stars\s+with\s+([\d,]+)\s+reviews?`. That is more stable than chasing a star-widget class, because screen-reader text changes less often than styling.

## Search pages, pagination and price cleanup

Best Buy search results live under a search page path that takes three parameters: `st` for the keyword (URL-encoded), `cp` for the page number (only added when it is above 1), and `sp` for sort order, with values like `-bestsellingsort` and `-Best-Discount`. Everything else on that page is decoration.

Prices from search cards need cleaning before they enter a database. Stripping non-numeric characters with something like `re.sub(r'[^\d.]', '', price_text)` handles the common case. Original prices are messier, since the regular-price block can render several elements and only one carries a dollar amount, so iterate and pick the one containing a currency symbol. A scraper that takes the raw card text without normalization will silently mix currency formatting into your price column.

## What Best Buy throws at you before any of this works

The blocking is real and the geo requirement is real. An open-source scraper configuration for Best Buy sets anti-scraping bypass on, forces country to US, and routes through a public residential proxy pool. That is the baseline combination: residential US IPs plus a bypass layer. Datacenter IPs from outside the US get served country-selection interstitials instead of products.

A Selenium-based scraper shows the same problem from the browser side. It waits up to 15 seconds for an element with class `us-link` to become clickable, clicks it, then sleeps 6 seconds before doing anything else. It also launches with `--no-sandbox` and `--disable-blink-features=AutomationControlled`, and notably does not run headless by default.

Lazy loading is the second layer. One scraper scrolls in 400px steps with a 0.5-second pause, refreshing the page height each step, then jumps to the bottom and waits 3 seconds. Another drives scrolling through a rendering scenario with two rounds of infinite scroll and 2-second waits between them, alongside `render_js`, a 10-second rendering wait and auto-scroll enabled. Whichever you use, budget rendering time into your timeout, because a Best Buy category page is not finished loading when the HTML arrives.

With ScraperAPI the equivalent levers are the `render=true` parameter for JavaScript rendering, `country_code=us` for a US IP, `premium=true` where CAPTCHA solving and retries are needed, and `session_number` when you need the same IP across a sequence of requests. If a rendered page comes back missing elements, `wait_for_selector` tells the renderer to hold until a specific node appears. It only works when `render=true` is also set, otherwise it is ignored.

Rendered concurrency has its own ceiling: a default burst limit of 10 rendered requests per second. If each rendered request takes 25 seconds, steady-state concurrency can reach roughly 250, but the rate at which you can start them is capped. 👉 [See what JavaScript rendering adds to a request](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Keeping the bill down

Two costs stack on a Best Buy job: rendering and bot-protection bypass. The practical moves:

Render only the pages that need it. Product detail pages that carry the embedded JSON may not need a browser at all; listing pages usually do.

Set `max_cost` per scrape so a single hard URL cannot quietly burn through credits, and check the Domain Cost Estimator before you scale a new URL pattern.

Cache aggressively by SKU. Prices change, spec sheets do not. Refetching specifications every hour is pure waste.

Route by product, not by page. Skip buying options and FAQ blocks when you only need price and stock.

If you would rather not run any of this, the alternative is a managed actor that bills per run or per result instead of per credit. Apify's Best Buy actor reads the retailer's GraphQL API and returns specifications as structured pairs, with no API key and no headless browser on your side; its billing runs through platform usage, compute units plus proxy traffic, rather than a per-result fee. Apify's general e-commerce tool prices differently again, charging separately for actor start, listings scraped, details, and optional residential proxy, browser rendering and AI summary. Neither number tells you anything about ScraperAPI's pricing, but the shape of the choice is the same everywhere: you either pay per request or per result.

## Scheduling and spotting real promotions

Electronics pricing moves fast around promotional periods, and a daily crawl will miss most of a deal window. Schedule hourly during the windows you care about and back off to daily otherwise.

Do not compute discounts yourself. `onSale` plus `totalSavingsPercent` filters to genuine markdowns directly, which avoids the classic error of treating a price drop against a stale list price as a promotion.

Track `availability` alongside price. A SKU that flips to archived while your feed still shows a live price produces phantom comparisons, and `dotComDisplayStatus` tells you that before a customer does.

## When a scraping service is the wrong purchase

If you need ten product pages a week, a browser extension and a CSV will do the job for free. If you only need Best Buy's own catalog metadata and no promotional presentation detail, an official product API may cover it with less maintenance.

A scraping API earns its cost when you are pulling thousands of SKUs on a schedule, when you need the page as rendered rather than the API's subset, and when you would otherwise spend engineering time on proxy rotation and bypass maintenance. If your volume is low or your need is one-off, the free credits on any provider will tell you that within an afternoon.

## Before you buy

Confirm four things. First, run three real Best Buy URLs through the Domain Cost Estimator: one search page, one product page, one product page with rendering on. The gap between those numbers is your actual per-record cost. Second, check whether the product data you need is in the embedded JSON, because skipping rendering is the single biggest cost lever. Third, size the credit allowance against your hourly cadence times SKU count times the credits per request, not against a plan name that sounds big enough. Fourth, confirm the overage behavior you want: auto-upgrade moves you up a tier automatically when credits run out, pay-as-you-go keeps you on the same plan at a fixed per-credit rate up to a monthly cap you set, and only the higher tiers offer that second option.

Selection rules, in one place. Testing a URL or two: free tier, 1,000 credits, 5 concurrent connections. A single category refreshed hourly with price and stock only: a small paid tier, no rendering on product pages if the JSON payload is enough. Multiple categories with rendered listing pages and bot-protection bypass on every request: size up two tiers from whatever your estimator math suggests, because the +10 bot-protection credits and the 10 rendering credits compound per request. Deal-window monitoring where volume spikes unpredictably: a tier with pay-as-you-go and a monthly cap.

The rule that survives all of it: the plan matters less than the cost per URL. Get that number for Best Buy first, then pick the tier that covers it. 👉 [Compare ScraperAPI plans and start with 1,000 free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)
