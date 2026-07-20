# ScraperAPI Setup Complete Guide: From API Key to First Request — Python, Node.js, Proxy Mode, Async Batching, and Credit Cost Optimization Explained

If you've ever stared at a half-finished scraper project at 2 a.m., wondering why your requests keep getting blocked while your proxy bill quietly triples, you're in the right place. Most "scraperapi setup" tutorials online stop at "copy this curl command and you're done." That's the easy 5%. The harder 95% — choosing the right integration method, sizing your plan to your real workload, avoiding the credit-multiplier trap, and scaling past a few hundred pages — is what actually decides whether your project ships or stalls.

This guide walks through the full ScraperAPI setup journey end to end: grabbing your API key, picking between the API endpoint, the SDK, and proxy port, sending your first real request, understanding what each request actually costs, scaling with async batch jobs, and finally matching all of that to the right plan for your budget. Everything here is based on the current official documentation and pricing page, so the numbers and parameters are accurate as of today.

## Why "ScraperAPI Setup" Trips Up So Many New Users

The setup looks deceptively simple. You sign up, you copy an API key, you fire off a GET request, and HTML comes back. Done. Then you point it at Amazon, and suddenly your 100,000 credits vanish after 6,600 requests. Or you forget to disable SSL verification in proxy mode and spend an hour chasing cryptic handshake errors. Or you set your request timeout to 10 seconds and wonder why half your scrapes return 500s.

The pattern across nearly every ScraperAPI review and Reddit thread is the same: the API itself is genuinely easy to use. The friction lives in the assumptions people bring to it — that "1 credit = 1 request" (it doesn't), that one integration method fits all (it doesn't), and that the free trial's 5,000 credits map cleanly to 5,000 scrapes (only if your target is a plain HTML blog). A proper setup means making deliberate choices at each layer, not just getting a "hello world" request to return 200.

## What ScraperAPI Actually Is — A 60-Second Orientation

ScraperAPI is a web scraping API that handles the unglamorous infrastructure work — proxy rotation across 40+ million IPs in 50+ countries, headless browser rendering, CAPTCHA solving, retries — so you send one API call and get back a clean HTML or JSON response instead of babysitting your own scraping stack. You don't manage proxies, you don't rotate user agents, you don't write CAPTCHA bypass logic. The API does all of that behind a single endpoint.

There are three ways to talk to ScraperAPI, and which one you pick during setup determines the rest of your code shape:

- **API Endpoint mode** — you hit `https://api.scraperapi.com` directly with your API key as a query parameter. Best for greenfield projects where ScraperAPI is your only data source.
- **SDK mode** — install the official SDK for Python, Node.js, PHP, Ruby, or Java, and call a client object. Best when you want retry logic and parameter handling baked in.
- **Proxy Port mode** — point any existing HTTP client at `proxy-server.scraperapi.com:8001` with `scraperapi` as the username and your API key as the password. Best when you're retrofitting ScraperAPI into a codebase that already uses a proxy pool.

All three modes expose the same parameters and the same credit costs. They're just different doors into the same room. 👉 [Start your free ScraperAPI trial with 5,000 API credits](https://www.scraperapi.com/?fp_ref=coupons)

## ScraperAPI Setup: Step 1 — Grab Your API Key

Register an account and head to your dashboard. The API key is displayed in multiple places throughout the dashboard for easy access — copy it once and store it somewhere safe (an environment variable, never hardcoded in source control).

A few important details:

- New accounts get **1,000 free API credits per month** with up to **5 concurrent connections**, just for signing up. No credit card required.
- On top of that, a **7-day trial** bumps you up to **5,000 one-time API credits** — enough to actually test against your real scraping targets instead of a toy example page.
- You can regenerate the API key from the dashboard if you suspect it's been compromised, but only **once every 24 hours**.

That trial window is the single most useful part of setup. Don't waste it on `example.com`. Point it at the actual domain you plan to scrape at scale, watch how many credits each request burns in the dashboard, and use that real number to pick a plan. A 5,000-credit trial against a plain blog gets you 5,000 test requests; the same trial against Amazon with rendering enabled gets you a few hundred. Run your own numbers before committing to anything.

## ScraperAPI Setup: Step 2 — Choose Your Integration Method

### Method A: The API Endpoint (simplest)

You send a `GET` request to `https://api.scraperapi.com` with two mandatory parameters: `api_key` and `url`. The API returns the HTML of the target URL.

python
import requests

target_url = 'https://www.example.com'
api_key = 'API_KEY'

request_url = f'https://api.scraperapi.com?api_key={api_key}&url={target_url}'
response = requests.get(request_url)

print(response.text)


javascript
import request from 'node-fetch';

const url = 'http://api.scraperapi.com/?api_key=API_KEY&url=https://example.com/';

request(url)
  .then(response => console.log(response))
  .catch(error => console.error(error));


One rule that bites people: **always put ScraperAPI parameters before the `url` parameter** in the query string. Otherwise they conflict with parameters that already exist on the target URL.

### Method B: The SDK (least code, retry logic included)

The SDK wraps the API endpoint in a client object and bakes in retry handling by default (3 retries out of the box, overridable with `retry=5` or whatever you need). Available for Python, Node.js, PHP, Ruby, and Java.

Install:

bash
pip install scraperapi-sdk


bash
npm install scraperapi-sdk --save


Use:

python
from scraperapi_sdk import ScraperAPIClient
client = ScraperAPIClient('API_KEY')
result = client.get(url='https://example.com/')
print(result)


javascript
import ScraperAPIClient from 'scraperapi-sdk';
const client = new ScraperAPIClient('API_KEY');

client.get('https://example.com/')
  .then(response => console.log(response))
  .catch(error => console.error(error));


To enable extra functionality, pass parameters as an additional argument:

python
result = client.get(url='https://example.com/', params={'render': True, 'premium': True})


### Method C: The Proxy Port (drop-in replacement for existing proxy pools)

If your codebase already speaks HTTP proxies, you can drop ScraperAPI in without touching request logic. The username is `scraperapi` and the password is your API key:

python
import requests

proxies = {
  "http": "http://scraperapi:API_KEY@proxy-server.scraperapi.com:8001"
}

r = requests.get('https://www.example.com', proxies=proxies, verify=False)
print(r.text)


To enable extra parameters in proxy mode, append them to the username separated by dots — e.g. `scraperapi.render=true.country_code=us`. SSL verification must be disabled (or you need to install ScraperAPI's CA certificate manually). This is the single most common setup mistake: forgetting `verify=False` and chasing handshake errors for an hour.

## ScraperAPI Setup: Step 3 — Send Your First Real Request

Now take it past hello-world. A practical setup includes retry handling and concurrency. Here's a Python pattern that uses `concurrent.futures` to parallelize requests, retries failed scrapes up to 3 times, and parses the result with BeautifulSoup:

python
import requests
from bs4 import BeautifulSoup
import concurrent.futures
from urllib.parse import urlencode

API_KEY = 'INSERT_API_KEY_HERE'
NUM_RETRIES = 3
NUM_THREADS = 5

list_of_urls = [
    'http://quotes.toscrape.com/page/1/',
    'http://quotes.toscrape.com/page/2/',
]

scraped_quotes = []

def scrape_url(url):
    params = {'api_key': API_KEY, 'url': url}
    for _ in range(NUM_RETRIES):
        try:
            response = requests.get('http://api.scraperapi.com/', params=urlencode(params))
            if response.status_code in [200, 404]:
                break
        except requests.exceptions.ConnectionError:
            response = ''

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, "html.parser")
        for quote_block in soup.find_all('div', class_="quote"):
            quote = quote_block.find('span', class_='text').text
            author = quote_block.find('small', class_='author').text
            scraped_quotes.append({'quote': quote, 'author': author})

with concurrent.futures.ThreadPoolExecutor(max_workers=NUM_THREADS) as executor:
    executor.map(scrape_url, list_of_urls)

print(scraped_quotes)


Two settings that matter more than people realize:

- **Timeout**: don't set one, or set it to at least 60 seconds. ScraperAPI auto-retries with a different proxy/header configuration for up to 60 seconds before returning a 500. A 10-second timeout on your side will kill perfectly good requests mid-retry.
- **Request size**: there's a 2MB limit per request. Fine for HTML, but if you're scraping PDFs or images, plan around it.

## ScraperAPI Setup: Step 4 — Understand Credit Costs Before Scaling

This is the step most tutorials skip and most users regret skipping. Every request consumes API credits, and the number of credits per request depends on the **domain** and the **parameters** you use. A "100,000 credits" plan does not mean 100,000 requests.

**Domain-based credit costs:**

| Domain Type | Credits per Request |
| --- | --- |
| Normal (flat) requests | 1 |
| E-Commerce (Amazon) | 5 |
| SERP (Google, Bing + subdomains) | 25 |
| Social Media (LinkedIn) | 30 |
| Cloudflare / Datadome / PerimeterX bypass | +10 |

**Parameter-based extra costs:**

| Parameter | Extra Credits per Request |
| --- | --- |
| `premium=true` (residential/mobile IPs) | +10 |
| `render=true` (JS rendering) | +10 |
| `screenshot=true` | +10 |
| `ultra_premium=true` (advanced bypass) | +30 |
| `premium=true` + `render=true` (combined) | 25 total |
| `ultra_premium=true` + `render=true` (combined) | 75 total |

Parameters with **no extra cost**: `wait_for_selector`, `country_code`, `session_number`, `device_type`, `output_format`, `keep_headers`, `autoparse`, `follow_redirect`.

One genuinely fair detail: **you're only billed for successful requests** (status codes 200 and 404). Failed scrapes don't burn credits, so you're not paying for the service's own failures — only for data it actually delivered.

Before scraping at scale, run your target URLs through the Domain Multiplier in the dashboard, or call the cost API endpoint directly:


https://api.scraperapi.com/account/urlcost?api_key=API_KEY&url=https://www.wikipedia.org/&render=true


Every response also includes a `sa-credit-cost` header showing exactly what that request cost. Use it during setup to sanity-check your math.

## ScraperAPI Setup: Step 5 — Scale Up With Async Batch Jobs

The synchronous API is fine for hundreds of URLs. When you start thinking in terms of tens of thousands, switch to the Async API endpoint at `https://async.scraperapi.com/batchjobs`. You submit a batch of up to **50,000 URLs per request** and receive a `statusUrl` for each, which you poll to collect results once the jobs finish.

python
import requests

r = requests.post(
    url='https://async.scraperapi.com/batchjobs',
    json={
        'apiKey': 'API_KEY',
        'urls': [
            'https://wikipedia.org/wiki/Cowboy_boot',
            'https://wikipedia.org/wiki/Web_scraping'
        ]
    }
)

print(r.text)


The response returns one job entry per URL with an `id`, `status`, and `statusUrl`:

json
[
  {
    "id": "04888c53-e322-4976-969d-8f8b39f016da",
    "status": "running",
    "statusUrl": "https://async.scraperapi.com/jobs/04888c53-e322-4976-969d-8f8b39f016da",
    "url": "https://wikipedia.org/wiki/Cowboy_boot"
  }
]


If your workload exceeds 50,000 URLs, split it into multiple batches — that's the hard ceiling per job, by design, for stability. Async mode also supports all the same parameters (`render`, `country_code`, `premium`, etc.) and the same structured data endpoints as the Sync API. It's the right choice once your setup moves from "test script" to "production pipeline." 👉 [Open ScraperAPI and start a free trial](https://www.scraperapi.com/?fp_ref=coupons)

## The Full ScraperAPI Plans Table — Which One Fits Your Setup?

All plans include JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA/anti-bot bypass, custom sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. The differences come down to volume, concurrency, geotargeting scope, and whether pay-as-you-go overflow is available.

| Plan | Monthly Price | Annual (per mo, 10% off) | API Credits / Month | Concurrent Threads | Geotargeting | Get Started |
| --- | --- | --- | --- | --- | --- | --- |
| **Free Trial** (7 days) | $0 | — | 5,000 (one-time) | 5 | — |  [Start free trial, no card needed](https://www.scraperapi.com/?fp_ref=coupons) |
| **Free Plan** (after trial) | $0 | — | 1,000 / month | 5 | — |  [Sign up](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49 | $44.10 | 100,000 | 20 | US & EU only |  [Get the Hobby plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Startup** | $149 | $134.10 | 1,000,000 | 50 | US & EU only |  [Get the Startup plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Business** | $299 | $269.10 | 3,000,000 | 100 | Global |  [Get the Business plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Scaling** (Most Popular) | $475 | $427.50 | 5,000,000 | 200 | Global |  [Get the Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Professional** | $975 | $877.50 | 10,500,000 | 300 | Global |  [Get the Professional plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Advanced** | $1,975 | $1,777.50 | 21,500,000 | 500 | Global |  [Get the Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22,000,000+ | 500+ | Global |  [Contact sales for Enterprise pricing](https://www.scraperapi.com/?fp_ref=coupons) |

A few non-obvious things to factor into your setup decision:

- **Geotargeting is gated by tier.** Hobby and Startup are US & EU only. If your target site blocks non-US IPs or you need country-level targeting elsewhere, you need at least Business.
- **Pay-as-you-go overflow starts at Scaling.** On Hobby, Startup, and Business, running out of credits mid-cycle means upgrading or contacting support. Scaling and above let you keep scraping at a fixed PAYG rate.
- **Credits don't roll over.** Whatever you don't use resets at renewal. Match your plan size to your real monthly volume rather than overbuying "just in case."
- **Unlimited analytics history kicks in at Business.** Hobby and Startup are capped at 30 days of dashboard history.
- **Priority support starts at Professional.**

**Which plan to pick after setup:**

- **Hobby ($49/mo)** — personal projects, side hustles, prototypes, monitoring a handful of pages. 100,000 credits stretches a long way on plain HTML; not so far on Amazon with rendering.
- **Startup ($149/mo)** — small SaaS or agency running scraping for a few clients. Real volume step-up, still US/EU only.
- **Business ($299/mo)** — first tier with global geotargeting and unlimited analytics. The right call if your scraper feeds production infrastructure.
- **Scaling ($475/mo) and above** — when "which plan" stops being the question and "how do I keep this predictable at high volume" becomes the question. PAYG overflow and priority support kick in here.

## Active ScraperAPI Promo Codes and Discounts

A few verified ways to reduce your setup cost:

- **Annual billing discount** — automatic 10% off every plan when you choose annual billing instead of monthly. No code needed, applied at checkout. This is the most reliable saving.
- **`START10`** — 10% off your first month on a subscription. Reported active across multiple coupon aggregators.
- **`ANWAR10`** — 10% off, applied at checkout. Reported active.
- **`rohit44`** — 10% off monthly subscriptions, mentioned in user forums.

Promo codes come and go. The annual billing discount is permanent and stacks cleanly with most introductory offers, so if you've validated the service during the 7-day trial and you're confident you'll use it for a year, annual is the safer bet. 👉 [Check current sign-up offers and start your free trial](https://www.scraperapi.com/?fp_ref=coupons)

## Common ScraperAPI Setup Mistakes (and How to Fix Them)

- **Setting a 10-second request timeout.** Fix: don't set a timeout, or set it to at least 60 seconds. ScraperAPI retries internally for up to 60 seconds before returning a 500.
- **Forgetting `verify=False` in proxy mode.** Fix: disable SSL verification, or install ScraperAPI's CA certificate (`https://api.scraperapi.com/proxyca.pem`) into your system trust store if you need verified SSL.
- **No retry logic in API Endpoint mode.** Fix: wrap requests in a 3-retry loop checking for 200/404 status codes. The SDK does this automatically; raw API calls don't.
- **Reading "100,000 credits" as "100,000 requests."** Fix: run your real target URLs through the dashboard's Domain Multiplier or the `urlcost` API endpoint before picking a plan.
- **Single-threaded scraping.** Fix: use `concurrent.futures.ThreadPoolExecutor` (Python) or `Promise.all` (Node.js) to parallelize up to your plan's concurrent thread limit. The difference between 1 and 20 threads is roughly 20x throughput, with no extra credit cost.
- **Ordering parameters wrong.** Fix: ScraperAPI parameters must come before the `url` parameter in the query string, otherwise they collide with parameters already on the target URL.
- **Not using `autoparse=true` on supported domains.** Fix: ScraperAPI has structured data endpoints for sites like Amazon and Google that return clean JSON instead of raw HTML — same credit cost, way less parsing code.

## ScraperAPI Setup FAQ

**Does one API request always cost one credit?**
No. The base rate is 1 credit for a standard page, but the domain (Amazon, Google, LinkedIn) and any parameters you add (rendering, premium proxies, ultra_premium) multiply that cost. Use the dashboard's cost estimator or the `urlcost` API endpoint before scraping at scale.

**Can I use ScraperAPI for free during setup?**
Yes. New accounts get 1,000 free API credits per month with 5 concurrent connections, no credit card required. A 7-day trial bumps you to 5,000 one-time credits — enough to test against real targets before paying.

**What happens if I run out of credits mid-month?**
On Hobby, Startup, and Business, you upgrade to the next tier or contact support. On Scaling, Professional, Advanced, and Enterprise, pay-as-you-go overflow billing kicks in at a fixed rate.

**Do unused credits roll over?**
No. Your credit balance resets at each renewal, so size your plan to your actual monthly usage rather than stockpiling.

**Can I cancel anytime?**
Yes, cancellation is available anytime from the dashboard or by contacting support, and you won't be charged again after cancelling. There's also a 7-day no-questions-asked refund policy.

**Which integration method should I pick during setup?**
- New project, ScraperAPI is your only data source → API Endpoint
- You want retry logic and parameter handling baked in → SDK
- Retrofitting ScraperAPI into existing proxy-based code → Proxy Port

All three expose the same parameters and the same credit costs.

**How many concurrent threads do I get?**
Free: 5. Hobby: 20. Startup: 50. Business: 100. Scaling: 200. Professional: 300. Advanced: 500. Enterprise: 500+.

## Bottom Line: Where to Start Your ScraperAPI Setup Today

The setup itself is genuinely fast — sign up, copy your API key, fire off a request, get HTML back. The part that takes longer, and the part that decides whether your project scales or stalls, is making deliberate choices at each layer: which integration method fits your codebase, what your real per-request credit cost is going to be, whether you need global geotargeting, and whether your volume justifies a pay-as-you-go tier.

The cleanest path through all of that is to test before you commit. Sign up for the free trial, point the API at your real targets, watch your credit consumption in the dashboard, then match what you observed to the plan table above. That's worth more than any tutorial, including this one.

👉 [Start your free ScraperAPI trial — 5,000 API credits, no credit card required](https://www.scraperapi.com/?fp_ref=coupons)
