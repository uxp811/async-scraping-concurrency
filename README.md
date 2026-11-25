# Async web scraping Python: how much faster is it, how many concurrent requests should you run, and when does a managed API beat your retry loop?

The whole pattern fits in a few lines: one `aiohttp.ClientSession`, a list of coroutines, one `asyncio.gather`. The number everyone quotes comes from Oxylabs' tutorial, which ran the same job twice on the same machine: **about 3 seconds** with `aiohttp` and `asyncio`, **almost 16 seconds** with plain `requests`. That gap was real for that URL list on that machine. It is not a multiplier you can count on, because a target that throttles you will drag both versions down to the same crawl rate.

python
import asyncio, aiohttp

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [scrape(session, url) for url in urls]
        await asyncio.gather(*tasks)

asyncio.run(main())


Below is the full loop with parsing and saving, then the part most tutorials skip: how to pick a concurrency number instead of guessing, how to read timeouts with p50 and p95 latency, how to stay off blocklists, and the point where a hosted scraper costs less than the week you'd spend on retries.

> **Quick read:** if your scraper is fine on speed but keeps returning 403s or empty HTML, the managed path starts free: 1,000 credits and 5 concurrent threads per month on the free plan, or a 7-day trial with 5,000 credits and no card. Hobby (100,000 credits, 20 threads) covers a single project. Business runs **$299/month**, or **$269.10/month** billed annually.
> 👉 [Compare all seven plans and current rates](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every plan, and what one credit actually buys

ScraperAPI does not price per page. It prices per credit, and the credit cost of a request depends on the domain and the parameters you send. That distinction decides which tier you need, so both tables matter.

| Plan | Credits / month | Concurrent threads | Price | Get it |
| --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0 | [ Start on the free plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | See pricing page | [ Check Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | See pricing page | [ Check Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | $299/mo, or $269.10/mo annual | [ Check Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | See pricing page | [ Check Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | See pricing page | [ Check Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | See pricing page | [ Check Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Custom quote | [ Talk to sales about Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons) |

The pricing page has a monthly/yearly toggle, and yearly takes 10% off. Business at $299 drops to $269.10 on annual billing, which works out to $3,229.20 for the year against $3,588 monthly, a $358.80 difference. Geotargeting widens as you climb: US and EU on Free, Hobby and Startup, country-level from Business upward. Analytics history goes from 30 days on the two entry tiers to unlimited from Business.

Now the credit table, which is where plan sizing gets interesting:

| Request type | Credits | Notes |
| --- | --- | --- |
| Standard page | 1 | The baseline |
| Amazon, Walmart, eBay | 5 | Per request |
| Google, Bing (incl. subdomains) | 25 | Per request |
| LinkedIn | 30 | Per request |
| Bot-protection bypass | 10 | Cloudflare, DataDome, PerimeterX/HUMAN, Akamai |
| JS rendering (`render=true`) | 10 | Per request |
| Premium proxies (`premium=true`) | 10 | Per request |
| Screenshot (`screenshot=true`) | 10 | Per request |
| Premium proxy + JS rendering | 25 | Both flags together |
| Ultra premium (`ultra_premium=true`) | 30 | Paid plans only |
| Ultra premium + JS rendering | 75 | Paid plans only |
| Geotargeting, sessions, device type, output format, auto parsing | 0 | No extra cost |

Run the division before you pick a tier. Hobby's 100,000 credits is 100,000 ordinary pages, but only 20,000 Amazon pages, 4,000 Google SERPs, or roughly 1,333 requests that need ultra premium plus rendering. Business at 3,000,000 credits is 600,000 Amazon pages or 120,000 SERPs. A plan that looks generous on the pricing page can turn out to be a tenth of that for the domain you actually care about.

Two things make this predictable rather than guesswork. Every response carries its cost in the `sa-credit-cost` header, and the dashboard has a Domain Cost Estimator you can point at a URL before scraping it. You can also set `max_cost` per request, so a single expensive page can't quietly eat your month.

## Why the synchronous version wastes your afternoon

`requests` does one thing at a time: send, wait for the body, parse, next URL. The waiting is dead time for your process, because the only thing happening is a socket receiving bytes. Oxylabs' comparison makes the cost visible: the sync script needed almost 16 seconds for the same workload that took about 3 async. Their own framing puts the async advantage at time efficiency, not magic.

python
def scrape(url):
    response = requests.get(url)
    soup = BeautifulSoup(response.content, "html.parser")
    book_name = soup.select_one(".product_main").h1.text
    rows = soup.select(".table.table-striped tr")
    product_info = {row.th.text: row.td.text for row in rows}
    save_product(book_name, product_info)


Twenty URLs at 700 ms each is 14 seconds of pure waiting. Nothing about your CPU is busy during that time.

## The async version, in the order you'd write it

`asyncio` gives you four moving parts, and only one of them is new syntax. The event loop schedules and switches tasks. Coroutines are functions declared with `async` that can pause and resume. Futures hold a result that isn't ready. Tasks wrap a coroutine so the loop can run it. You mostly touch tasks.

The structure below follows the tutorial's six steps: a main coroutine, a timer, a task list, a scrape function, a save function, and a run call.

python
import asyncio, time, json, aiohttp
from bs4 import BeautifulSoup

def save_product(book_name, product_info):
    filename = book_name.replace(" ", "_")
    with open(f"data/{filename}.json", "w") as f:
        json.dump(product_info, f)

async def scrape(session, url):
    async with session.get(url) as resp:
        body = await resp.text()
    soup = BeautifulSoup(body, "html.parser")
    book_name = soup.select_one(".product_main").h1.text
    rows = soup.select(".table.table-striped tr")
    product_info = {row.th.text: row.td.text for row in rows}
    save_product(book_name, product_info)

async def main():                      # step 1
    start_time = time.time()           # step 2
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(scrape(session, url)) for url in urls]   # step 3
        await asyncio.gather(*tasks)
    time_difference = time.time() - start_time
    print("Scraping time: %.2f seconds." % time_difference)

asyncio.run(main())                    # step 6


The `async with session.get(url) as resp` block plus `await resp.text()` is the whole trick. `await` hands control back to the loop, which starts the next request while the first response is still in flight. `asyncio.gather` waits for all of them.

Two housekeeping details. Create a `data` folder next to the script before running, or `save_product` has nowhere to write. And the tutorial wraps `main()` with `loop = asyncio.get_event_loop()` followed by `loop.run_until_complete(main())`, which keeps the script alive until the coroutine finishes; `asyncio.run(main())` does the same job and is shorter.

## How many concurrent requests should you actually run?

Start lower than you think, measure, then raise. The most useful public case study on this comes from a developer running a three-level crawl with `httpx.AsyncClient`, a task queue and a pool of workers. He settled on **20 workers**, and said why: raising the number produced a lot of `httpx.ReadTimeout` errors. He couldn't explain it, and offered only a guess that parsing was holding the event loop long enough that responses missed the timeout window. More workers made his crawler slower and noisier, not faster.

The structural fix is to stop thinking in fixed worker pools. Instead of N workers pulling from a queue, start a task per URL and cap how many are in flight with a semaphore:

python
sem = asyncio.Semaphore(10)

async def bounded(session, url):
    async with sem:
        html = await fetch(session, url)
    return parse(html)   # the slot is free again while you parse


Parsing happens after the network slot is released, so a slow BeautifulSoup pass on one page doesn't block a request that could already be running.

One detail about coroutines that bites people coming from threads: a coroutine is a future-like object, and calling it begins execution immediately, running up to its first `await`. If you build a big list of coroutine objects in one go, they all start. Wrapping each call in a plain lambda, or scheduling them as tasks under a semaphore, keeps the start timing under your control. That is the difference between a work-in-progress limit and a wish.

> There is a coincidence worth noticing: the worker count that developer arrived at by trial and error, 20, is the same concurrency ScraperAPI puts on its Hobby tier.

## Timeouts: measure p50 and p95 before you change the number

Record how long each task takes, then look at the distribution rather than the average. Time each `await` call and log the samples. The median tells you the typical latency; the 95th percentile tells you what your worst normal case looks like. Try a few concurrency settings and watch how both move. The developer whose scraper was hitting timeouts had a 10-second limit configured, and the advice given was to keep the measured latency well below it, not to raise the limit.

Three things cause those timeouts, and only one is your code. The target server is rate-limiting you. The network is dropping packets or your bandwidth is saturated. Or a coroutine is doing blocking work, a synchronous HTTP call, a heavy parse, a blocking database write, and stalling the event loop for everyone else. There's also a capacity rule that's easy to miss: you should never have more active workers than available connections.

When p95 sits close to your timeout, adding workers makes it worse. Cutting the pool in half and re-measuring usually tells you more than another round of tuning ever will.

## Errors, delays, and staying off blocklists

Handle failures inside the coroutine, and make the success check explicit. A response object isn't a page until the status says so.

python
async def fetch(session, url):
    try:
        async with session.get(url) as resp:
            if resp.status == 200:
                return await resp.text()
            print(f"Error: {resp.status} for {url}")
            return None
    except aiohttp.ClientError as e:
        print(f"Error: {e} for {url}")
        return None
    finally:
        await asyncio.sleep(random.uniform(0.5, 1.5))


That `finally` block adds a random half-second to a second and a half to every request, successful or not. It directly reduces the speedup you just built. That's the trade.

Per-domain politeness is stronger than a global delay, because a global sleep punishes the fast sites. Keep a dictionary of the last request timestamp per host and wait out the gap:

python
last_hit = {}
delay = 2.0

async def polite_get(session, url):
    host = urlparse(url).netloc
    wait = delay - (time.monotonic() - last_hit.get(host, 0))
    if wait > 0:
        await asyncio.sleep(wait)
    last_hit[host] = time.monotonic()
    async with session.get(url) as resp:
        return await resp.text()


Use `time.monotonic()` for this, never wall-clock time. Follow `robots.txt` when you crawl links, and keep a realistic User-Agent. Ignoring those two is how IPs end up on a firewall rule.

## When maintaining the scraper stops paying

Add up what the politeness rules cost you. At one request every couple of seconds per domain, a 50,000-page crawl of a single site takes more than a day no matter how elegant your event loop is. Now add a proxy pool to rotate, JS rendering for the pages that ship an empty `<div>`, and a retry layer for the ones that fail halfway. The async code isn't the expensive part anymore.

ScraperAPI's answer to that is its Async API, and it's a different shape from what you've been building. You POST a job with your API key and target URL, and get back a job id plus a status URL you can poll; results can also be pushed to a webhook, which removes polling entirely.

The constraints are the interesting part. A single batch job accepts up to **50,000 URLs**. A job keeps retrying until the pages come back successfully or **24 hours** pass. Results are stored for up to **72 hours**, with 24 hours guaranteed, after which they're deleted and you resubmit. Cost control runs through a `max_cost` parameter in `apiParams`; if the job would exceed it, you get a 403 instead of a bill. On the synchronous side, the concurrency ceiling is enforced with a 429 that asks you to come back within your plan's limit. Requests over the line are turned away, not queued, and you're not charged for them.

Compare that to your own scraper's failure mode, where too many workers produce `ReadTimeout` exceptions and a queue full of half-finished jobs. Getting told "no" at the door is easier to work with.

👉 [Start the 7-day trial with 5,000 credits, no card required](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Questions that come up before switching

**Is async always faster?** No. It removes idle waiting on network I/O. It does nothing for a server that limits you to one request every few seconds, and it makes things worse when you push concurrency past what the target or your connection tolerates.

**aiohttp or httpx?** Both are async clients and both work with `asyncio.gather`. The tutorial used `aiohttp`; the timeout case study used `httpx.AsyncClient`. Pick one, then watch for blocking calls inside your coroutines, which hurt either way.

**What happens when credits run out mid-month?** Scraping pauses until they renew, by default. On Hobby through Professional you can switch on auto-upgrade, which moves you to the next tier when credits are exhausted; it's opt-in. Scaling, Professional, Advanced and Enterprise can use pay-as-you-go at a fixed per-credit rate up to a monthly cap you set.

**Do I still need my own proxies?** Only if you stay self-hosted. Through the API, premium proxies cost 10 credits per request and ultra premium 30, with bot-protection bypass also at 10. Rotating a pool yourself is the part most people are happiest to hand off.

**Can I keep using my async code with the API?** Yes, in the sense that you can fire the job submissions concurrently from `asyncio` and gather the status URLs. The retrying and rendering happen on their side.

## Picking a tier without overbuying

Match the plan to the job, not to the credit number.

A single project scraping ordinary HTML, under a few hundred thousand pages a month, fits Hobby at 20 threads and 100,000 credits. Start on the free plan or the trial to measure your real cost per page first, because a domain that needs rendering plus premium proxies will burn credits ten to seventy-five times faster than a plain one. Business at $299/month makes sense when you're running production workloads across multiple domains and the 100 threads plus country-level geotargeting are doing actual work. Above that, Scaling and up are for volume, and the pay-as-you-go option matters more than the sticker price.

If your pages are static and your volume is small, the plain `aiohttp` script in this article plus a semaphore of 10 will do everything you need for free.

Before you enter card details, confirm two numbers on the live pricing page: the credit cost for your specific target domain, using the estimator, and whether annual billing is the right call for a project you might abandon in three months. The 10% annual discount is real, and so is the commitment.

👉 [Check the current tiers and pick the concurrency limit you actually need](https://www.scraperapi.com/pricing?fp_ref=coupons)
