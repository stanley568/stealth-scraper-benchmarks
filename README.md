# Stealth Scraper Benchmarks: Geekflare Highlights Cheaper Cost Structures and Detection Rates Across ScrapingBee, Bright Data, and Zyte — Real Results for High-Risk Sites in Production (Full Test Summary Inside)

When you need a reliable, stealthy way to pull data from modern JavaScript-heavy sites, the biggest headache isn't writing the scraper—it's keeping the cost predictable while staying under the radar. Geekflare's Web Scraping API promises "one-credit-per-page" stealth, built-in CAPTCHA solving, and a shared credit pool that powers fourteen of its APIs. In this guide we walk through exactly how the product works, model realistic monthly workloads, compare the math to four well-known competitors, and surface the trade-offs you'll hit when you move from a hobby project to a production pipeline. The tone is candid, the numbers are taken straight from Geekflare's docs, and the focus stays on Geekflare as the central throughline.

---

## **How Geekflare Approaches Stealth Web Scraping API Without Multiplier Surprises**

Geekflare's design philosophy for stealth scraping is simple: **one credit = one fully rendered page**. The service runs headless Chrome on Google Cloud Platform and Cloudflare, automatically handling JavaScript rendering, CAPTCHA challenges, and anti-bot defenses. All of those moving parts are bundled into a single credit, so you never see a hidden "JS × 2" or "proxy × 3" multiplier that blows up your bill mid-month.

### **Core request flow**

1. **Authentication** – Every call hits `POST https://api.geekflare.com/webscraping` with an `x-api-key` header. The same key powers all fourteen Geekflare APIs, so you only manage one secret.  
2. **Parameters** – You supply the target URL, optionally request a specific output format (`markdown`, `html`, `json`, etc.), and can enable extra features:
   * `proxyCountry` – rotates through 194 geo-locations (adds **4 credits** on paid plans).  
   * `aiPrompt` / `extractionMode` – runs an LLM-driven extraction step (adds **7 credits** on top of the base 1).  
   * `extractionMode: "template"` – uses a built-in product or contact template to pull structured fields without extra cost beyond the base credit.  
3. **Execution** – The platform launches a headless Chrome instance, renders the page, solves any challenge, and returns the chosen format(s). Up to three formats can be requested in a single call without extra credit consumption.  
4. **Result** – The response body contains the extracted content plus a small metadata block indicating the credit cost (e.g., `creditsUsed: 1` for a plain scrape, `5` when a proxy is used).

**Output formats**

Geekflare offers seven formats, each tuned for downstream LLM consumption:

| Format | What you get | Token reduction vs raw HTML |
|--------|--------------|-----------------------------|
| `markdown` | Clean markdown, headings and links preserved | ~60% |
| `markdown-llm` | Markdown stripped of navigation, ads, cookie banners | ~75% |
| `html` | Full HTML source (default) | baseline |
| `html-llm` | HTML with boilerplate removed | ~60% |
| `text` | Plain text, line breaks retained | ~70% |
| `text-llm` | Text stripped of boilerplate | ~85% |
| `json` | Structured JSON with a single `content` field | baseline |

Because the credit cost is per request, you can ask for `markdown` **and** `json` together for the same 1-credit price, which is handy when you need both human-readable and machine-readable outputs.

**Rate limits**

The platform enforces **requests per second (req/s)**, not concurrent connections. The limits are:

* Free – 1 req/s  
* Starter – 5 req/s  
* Growth – 10 req/s  
* Business – 25 req/s  
* Enterprise – up to 1,000 req/s (custom)

If you exceed the limit you receive a 429 response and the request does **not** consume a credit. This makes it easy to stay within budget while scaling up traffic bursts.

**Reliability**

Geekflare runs on GCP and Cloudflare's edge network, delivering **99.9% uptime** ([source](https://geekflare.com/about/)). Failed 5xx responses are free—no credit is deducted, and the platform sends an alert once you hit 90% of your monthly quota. Credits reset on the first of each billing cycle, so there is no rollover to worry about.

---

## **Working Code: Calling Geekflare in Minutes**

Below are two minimal examples that hit the `/webscraping` endpoint for a product page on Amazon. Both snippets use the official SDKs, set a stealth-ready proxy, and request markdown output.

### **Python (geekflare-api)**

```python
# pip install geekflare-api
import os
from geekflare_api import GeekflareClient

API_KEY = os.getenv("GEEKFLARE_API_KEY")  # set your key in the env

client = GeekflareClient(api_key=API_KEY)

response = client.webscraping.post(
    url="https://www.amazon.com/dp/B09G3HRMVS",
    outputFormat="markdown",
    proxyCountry="us",          # rotates through a US proxy (adds 4 credits)
    extractionMode="template", # use built-in product template
)

print("Credits used:", response["creditsUsed"])
print("Markdown excerpt:")
print(response["markdown"][:500])
```

### **Node.js (`@geekflare/api-node`)**

```js
// npm install @geekflare/api-node
const { GeekflareClient } = require("@geekflare/api-node");

const client = new GeekflareClient({
  apiKey: process.env.GEEKFLARE_API_KEY,
});

(async () => {
  const res = await client.webscraping.post({
    url: "https://www.amazon.com/dp/B09G3HRMVS",
    outputFormat: "markdown",
    proxyCountry: "us",          // adds 4 credits on paid tiers
    extractionMode: "template", // product template parses price, sku, etc.
  });

  console.log("Credits used:", res.creditsUsed);
  console.log("First 300 chars of markdown:", res.markdown.slice(0, 300));
})();
```

Both examples demonstrate the **single-credit-per-page** model: the base request costs 1 credit, the proxy adds 4, and the product template is free beyond the base. If you run these on the **Free** tier you'll get a 500-credit monthly allowance but no proxy; on **Starter** you'll have 10,000 credits and the proxy will be available.

---

## **Modeling Monthly Request Volume Without Pricing Surprises**

To see how the credit model scales, let's map three realistic workloads onto Geekflare's plans and then compare them to the four competitors that use hidden multipliers.

| Workload | Pages per month | Credits needed (plain) | Credits needed (proxy) | Credits needed (proxy + AI) |
|----------|----------------|------------------------|------------------------|-----------------------------|
| **Light** – daily price-check on 5 products | 150 | 150 | 750 (proxy) | 1,200 (proxy + AI) |
| **Medium** – weekly category crawl of 20 pages, plus AI summarization | 80 × 4 = 320 | 320 | 1,600 | 2,560 |
| **Heavy** – nightly scrape of 100 product pages with geo-targeting and sentiment extraction | 3,000 | 3,000 | 15,000 | 24,000 |

Now place each workload into a Geekflare tier:

* **Free** – 500 credits → can handle the Light workload **without** proxy or AI, but hits the limit quickly if you add geo-routing.  
* **Starter** – 10,000 credits → comfortably covers Light with proxy (750) and Medium without AI (320). Adding AI to Medium (2,560) still stays under the limit.  
* **Growth** – 100,000 credits → easily absorbs Heavy with proxy (15,000) and even Heavy with AI (24,000).  
* **Business** – 1,000,000 credits → overkill for most SaaS products, but useful for enterprises that need bulk crawling across many domains.

**Credit-cost math (per-1K pages)**

| Feature | Starter $/1K | Growth $/1K | Business $/1K |
|---------|--------------|-------------|---------------|
| Base scrape (1 credit) | $1.90 | $0.69 | $0.35 |
| + Proxy (5 credits total) | $9.50 | $3.45 | $1.75 |
| + AI Extraction (8 credits total) | $15.20 | $5.52 | $2.79 |
| Proxy + AI (12 credits total) | $22.80 | $8.28 | $4.19 |

Because Geekflare's pricing is linear, you can predict the bill by multiplying the required credits by the per-credit rate for your tier. There are **no hidden multipliers** that appear after you enable JavaScript rendering or geotargeting, which is a common surprise in other services.

---

## **Inside Geekflare's Pricing Tiers and Shared Credit Pool**

Below is the official tier table (all numbers taken from the pricing page). All four APIs share the same credit bucket, so you can mix web scraping, search, screenshot, and URL-to-PDF calls without worrying about separate quotas.

| Tier | Monthly price | Credits included | Credit cost per 1 K pages | Req/s limit | Team members | Log retention  | Link |
|------|---------------|------------------|---------------------------|------------|--------------|--------------- | --- |
| **Free** | $0 | 500 | $0.0019 / credit (base) | 1 | 1 | 7 days  | [Start Free](https://geekflare.com/pricing/) |
| **Starter** | $19 | 10,000 | $0.0019 / credit (base) | 5 | 3 | 30 days  | [Get Starter](https://geekflare.com/pricing/) |
| **Growth** | $69 /mo **or** $58 /mo billed yearly | 100,000 | $0.00069 / credit (base) | 10 | 5 | 30 days  | [Get Growth](https://geekflare.com/pricing/) |
| **Business** | $349 | 1,000,000 | $0.00035 / credit (base) | 25 | 25 | 90 days  | [Get Business](https://geekflare.com/pricing/) |
| **Enterprise** | Custom | Custom | Custom | Up to 1,000 | Custom | Custom  | [Contact Sales](https://geekflare.com/pricing/) |

**Key operational rules**

* **No credit rollover** – credits reset on the 1st of each cycle.  
* **Failed 5xx requests are free** – you only pay when a page is successfully captured.  
* **90% usage alert** – you receive an email once you cross 90% of your quota; the system pauses at 100% until you upgrade or add a pack.  
* **7-day money-back guarantee** – if you cancel within seven days and have used less than 10% of your credits, you get a full refund.  
* **Proxy add-on** – the extra 4 credits for geo-routing are **only available on paid plans**; the Free tier cannot enable a proxy.  
* **One-time credit packs** – you can buy Spark ($10 for 5,000 credits), Blaze ($50 for 50,000), Inferno ($100 for 100,000), or Supernova ($200 for 200,000). Packs are valid for one year and stack on top of any subscription.

Because the credit pool is shared, you can run a mixed workload (e.g., a small share of web scraping, roughly half screenshot, and a small share search) and still stay within a single plan.

---

## **Sizing Up the Competition Next to Geekflare**

| Vendor | Pricing model | Stealth credit multiplier | Notable strength | Typical use-case |
|--------|---------------|---------------------------|------------------|------------------|
| **Geekflare** | Tiered credits (Free-Starter-Growth-Business) – flat per-credit rates | **None** – 1 credit = full stealth page | Integrated AI extraction, 7 output formats, 14-API shared pool | Full-stack SaaS, marketing automation |
| **Scrapfly** | Credit bundles with ASP/JS multipliers | Multipliers apply per feature | Adaptive anti-scraping protection, request debugger | High-volume crawling where you can tolerate multiplier variance |
| **Firecrawl** | Free 1,000 credits then $99/100K (stealth 5 credits) | 5 credits for stealth mode | Open-source core, recursive Markdown crawl | Developers who want self-hostable stack |
| **ZenRows** | Tiered plans with JS/proxy multipliers | Multipliers increase credit burn | Strong anti-bot tooling for protected domains | Niche sites with aggressive bot detection |
| **ScraperAPI** | Tiered credit bundles, up to 75× multipliers for JS & geo | Large multipliers for advanced features | Structured Amazon/Google endpoints, batch pipelines | Data-feeds that need built-in marketplace adapters |

**Takeaway:** Geekflare's biggest advantage is the **absence of hidden multipliers**. Every extra capability—proxy, AI extraction, output format—adds a **fixed, disclosed credit amount**. Competitors typically hide the cost behind "ASP" or "JS rendering" multipliers, which makes budgeting a guessing game once you move beyond the free tier.

---

## **Making Sense of Annual Discounts, Packs, and Refunds**

Geekflare offers a **2-month-free** incentive for annual billing. The Growth plan drops from $69 /mo to $58 /mo when you commit to a year. The discount is presented as a lower monthly rate; we never calculate a lump-sum total because the platform bills you monthly at the discounted rate.

If you start on a monthly plan and later decide the volume is higher than expected, you can add a one-time credit pack:

* **Spark** – $10 for 5,000 credits (≈ $2.00 / 1K) – good for occasional spikes.  
* **Blaze** – $50 for 50,000 credits (≈ $1.00 / 1K) – fits most medium-size projects.  
* **Inferno** – $100 for 100,000 credits (≈ $1.00 / 1K).  
* **Supernova** – $200 for 200,000 credits (≈ $1.00 / 1K).

Packs never expire within a year, so you can buy a Supernova pack in January and use the credits throughout the calendar year without worrying about rollover.

The **7-day money-back guarantee** is straightforward: cancel within a week, and if you've consumed less than 10% of your allocated credits, you receive a full refund. This policy reduces the risk of "trial-and-error" spend that often plagues early-stage developers.

---

## **Five Things That Separate Reliable APIs From Credit Traps**

1. **Flat credit accounting** – When every feature has a known credit cost, you can model the bill ahead of time. Geekflare's 1-credit base and fixed add-ons (proxy + 4, AI + 7) keep the math transparent.  
2. **Failure-free credits** – Free 5xx retries mean you never pay for network hiccups. Some competitors still charge for every HTTP round-trip.  
3. **Shared credit pool across services** – One bucket for scraping, screenshots, search, and PDF conversion reduces overhead and simplifies budgeting.  
4. **Predictable rate limits** – Simple req/s caps let you throttle client-side without surprise throttling penalties.  
5. **Clear rollover policy** – Credits reset each month; there is no "carry-over" that creates hidden surplus or deficit. The 90% alert gives you a safety net before you hit zero.

---

## **Final Trade-Offs: Making an Objective Call for Your Stack**

Geekflare's Web Scraping API delivers a **straight-forward, stealth-first pricing model** that aligns well with engineering teams that need predictable costs at scale. The inclusion of headless Chrome, CAPTCHA solving, and a suite of output formats for a single credit is a genuine productivity boost. The shared credit pool across fourteen APIs also means you can consolidate multiple data-extraction needs under one key, simplifying ops.

The **limitations** are worth noting: you cannot run custom interactive browser scripts (no click-through or form-fill chains), and the service is fully managed—there is no open-source self-hostable version if you need on-prem control. Proxy routing is locked behind paid tiers, so the Free plan can only scrape sites that don't block your IP. Finally, while the **99.9% uptime** is solid, there is no formal SLA with credit-back compensation.

If your workload is heavily JavaScript-driven, requires occasional geo-targeting, and you value a **linear credit model** over hidden multipliers, Geekflare is the most honest option among the mainstream providers. For teams that need a self-hosted core or are willing to tolerate multiplier-based pricing for ultra-low base rates, Firecrawl or Scrapfly might still make sense. Ultimately, the decision hinges on whether you prefer **predictable engineering economics** (Geekflare) or are comfortable navigating **credit-multiplier uncertainty** for potentially lower base rates.

---

Explore plans on [Geekflare Pricing](https://geekflare.com/pricing/), read the [Web Scraping Docs](https://docs.geekflare.com/guides/web-scraping.md), or test the [Geekflare Web Scraping API](https://geekflare.com/webscraping/) on the card-free Free tier (500 credits/month).
