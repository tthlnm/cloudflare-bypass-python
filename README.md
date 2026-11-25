# Bypass Cloudflare in Python: cfscrape vs cloudscraper, the two cookie rules behind most 403s, and what a protected page costs on a hosted API

A Cloudflare challenge in front of your Python scraper leaves two workable paths. Solve the JavaScript challenge locally with an open-source library, keep the cookies it returns, and replay them on every request. Or hand the URL to a hosted scraping API that does the challenge work on its own infrastructure, so your code stays one HTTP call.

The first path costs nothing but your time and works at small scale, provided you respect two rules that most tutorials skip. The second costs credits per request and removes the maintenance. What follows: the install commands and parameters that actually matter for both libraries, the cookie and IP constraints that explain most sudden 403s, and the per-request credit math for ScraperAPI so you can price the alternative against your own hours.

> **Quick pick.** Scraping a handful of pages now and then, on a machine where you can run `node -v`? Install cloudscraper and move on. Running on a schedule, at volume, or on a host where you can't install a JavaScript engine? Go hosted. On ScraperAPI a Cloudflare-protected page costs 11 credits (1 for the page, 10 for the bypass), so the free 1,000 credits cover roughly 90 protected requests before you pay anything. 👉 [Start on the free plan and see which tier matches your volume](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What actually changes between ScraperAPI's tiers

ScraperAPI builds its plans as a ladder, and the differences that decide your bill are structural rather than cosmetic. The two mechanisms to understand are **auto-upgrade**, which is opt-in and moves your account to the next plan the moment credits run out, and **pay-as-you-go**, which keeps you on your current plan at a fixed per-credit rate up to a monthly cap you set yourself. Auto-upgrade exists on Hobby, Startup, Business, Scaling and Professional. Pay-as-you-go exists on Scaling, Professional, Advanced and Enterprise. Credit allowances, concurrency limits and the monthly or annual price of each tier are listed on the pricing page, since they move with billing cycle and promotions.

| Plan | What happens when credits run out | Best fit | Start |
| --- | --- | --- | --- |
| Free | Scraping pauses until renewal | A first test against your own target URLs (1,000 credits, 5 concurrent connections) | [Claim the free credits](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Auto-upgrade available | One small project that hits a protected page occasionally | [Check the Hobby tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Auto-upgrade available | Regular jobs that have outgrown Hobby's allowance | [Check the Startup tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Auto-upgrade available | Several jobs running on a schedule | [Check the Business tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Auto-upgrade or pay-as-you-go | Workloads you'd rather not hard-cap | [Check the Scaling tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Auto-upgrade or pay-as-you-go | The same flexibility with a larger allowance | [Check the Professional tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Pay-as-you-go | High, sustained volume | [Check the Advanced tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Pay-as-you-go | Custom limits and terms | [Talk to ScraperAPI about Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons) |

One thing worth deciding before you buy: auto-upgrade is convenient and it is also how a $49 month becomes a $99 month without you clicking anything. Turn it on only if a stalled scrape costs you more than the next tier does.

## The free route, part 1: cfscrape

`pip install cfscrape` gives you a Requests-based module aimed at the classic "Checking your browser before accessing…" interstitial. Its documented support range is Python 2.6 through 3.7, and it needs Node.js 10 or above on the same machine, because the library passes Cloudflare's obfuscated challenge to Node instead of reimplementing the obfuscation itself.

The API surface is a two-line change from normal Requests code:

python
import cfscrape
scraper = cfscrape.create_scraper()
print(scraper.get("http://somesite.com").content)


`create_scraper()` returns a `CloudflareScraper`, which subclasses `requests.Session`. Every `requests.get()` and `requests.post()` you already have becomes `scraper.get()` and `scraper.post()`. Sites without Cloudflare are handled as ordinary requests, so you can point the same session object at everything.

Two behaviours surprise people. The first visit to any site with Cloudflare's anti-bot page sleeps for **5 seconds** while the challenge is answered; subsequent requests through that session don't. If a site is under heavy load and the challenge fails, `create_scraper(delay=10)` (or 15) raises the wait, and the README's own advice is not to touch that value unless an error tells you to.

The second is the token mechanism. Cloudflare issues two cookies, `cf_clearance` and `__cfduid`. `cfscrape.get_tokens()` returns them as a dict, `cfscrape.get_cookie_string()` returns a ready-made `Cookie` header, and both return a tuple of `(cookie, user_agent_string)`. Pass the same keyword arguments you would give Requests, including proxies:

python
tokens, user_agent = cfscrape.get_tokens(
    "http://somesite.com",
    proxies={"http": "http://localhost:8080", "https": "http://localhost:8080"},
)


## The free route, part 2: cloudscraper

`pip install cloudscraper` installs the other library in this family. Same idea, same shape of API (`cloudscraper.create_scraper()` returns a `CloudScraper` that subclasses `requests.Session`), different maintenance track. Its current release is **1.2.71, published 25 April 2023**, and it targets Python 3.x with Requests 2.9.2 or newer plus requests_toolbelt 0.9.1 or newer.

The JavaScript side is where it diverges. cloudscraper can drive ChakraCore, js2py, Node.js or V8 through Sony's v8eval, and its **default is a native pure-Python solver**, so a clean `pip install` runs without any external runtime. js2py ships as a dependency; the other engines you install yourself.

It also covers challenges cfscrape explicitly refuses. cfscrape's documentation is blunt about reCAPTCHA: "If there is a reCAPTCHA challenge, you're out of luck." cloudscraper instead integrates third-party solvers, and you supply the paid key:

python
scraper = cloudscraper.create_scraper(
    captcha={"provider": "anticaptcha", "api_key": "your_key"}
)


Supported providers are 2captcha, anticaptcha, capsolver, deathbycaptcha and 9kw, and the first two accept `no_proxy` so your proxy credentials don't leak to the solver. A "free" library that needs a captcha subscription per thousand solves is not free at that point, which is worth pricing in before you commit.

## The parameters worth knowing when a request still fails

cloudscraper exposes a set of knobs that map to real failure modes, and they're the fastest way to diagnose a stubborn target:

- `browser={'browser': 'chrome', 'platform': 'android', 'desktop': False}` narrows the user-agent pool to mobile Chrome on Android; `custom` looks your string up in `browsers.json` and reuses that browser's headers and cipher suite if there's a match.
- `ecdhCurve='secp384r1'` swaps the default `prime256v1` curve, which the docs present as a fix for handshake failures on servers that demand a stronger one.
- `server_hostname='www.somesite.com'` together with `headers={'Host': 'www.somesite.com'}` lets you request a backend host while presenting the front-end hostname for SNI.
- `disableCloudflareV1=True` skips the deprecated v1 path; `allow_brotli=False` turns off Brotli handling; `debug=True` prints the headers and body it's working with.

## The two rules that break most Python Cloudflare scripts

These are the reason a script that works on your laptop returns 403 on a server, and both come straight from the library documentation.

1. **The user-agent that fetched the token must be the user-agent that uses it.** `get_tokens()` hands back a `user_agent_string` alongside the cookies. If your downstream request, curl call or scraping tool sends a different UA, Cloudflare treats the session as a bot and the clearance is worthless.
2. **The IP that solved the challenge must be the IP that spends the token.** If you obtained `cf_clearance` through a proxy, every request carrying that token has to go back through the same proxy. Rotating proxies mid-session invalidates the cookie, which is exactly what happens when someone wires a token into a rotating proxy pool and can't work out why half the requests fail.

A corollary for session migration: `requests.session()` and `cfscrape.create_scraper()` are not interchangeable after the fact, because not all Requests session attributes transfer cleanly. If you're retrofitting an existing scraper, replace the session constructor rather than patching a live one.

## Where the free libraries stop

Both projects describe Cloudflare as something that changes periodically and both ask users to file an issue when the anti-bot page shifts, which tells you the maintenance model is reactive by design. The documentation in front of you describes the classic IUAM interstitial; neither README covers newer challenge formats, so treat "does this still work today on site X" as something you verify with one request rather than something the docs promise.

The version dates are the other signal. cloudscraper's last release is April 2023, cfscrape's stated support stops at Python 3.7, which is several Python releases behind the current interpreter. Neither fact proves the code is broken. Together they mean you're running unmaintained dependencies against a target that changes on its own schedule, and that's a risk you carry per site, per month.

## What changes when you hand the challenge to an API

With a hosted service, your Python code stops solving anything. You send the target URL and your key, and you get HTML back. ScraperAPI says it routes through residential IPs, handles TLS and JA3 fingerprinting, keeps cookies and headers consistent across a session, and retries the 520–524 range of origin errors automatically, which is the class of failure that looks like a block but isn't.

The pricing model is where this differs from a flat subscription. One credit buys a standard page. Amazon costs 5, Google and Bing cost 25 including subdomains, LinkedIn costs 30, and any site behind bot protection such as Cloudflare, Datadome or PerimeterX adds **10 credits per request** when the bypass runs. A dashboard tool called the Domain Cost Estimator gives you the exact figure for a URL, and the `max_cost` parameter caps what a single scrape is allowed to spend.

Two operational details matter more than the marketing copy. Exceeding your concurrency limit returns a 429 and the request is refused rather than queued, with no overage charge. And when credits run out, scraping pauses unless you've opted into auto-upgrade or pay-as-you-go.

## The credit math for a Cloudflare-heavy job

Take a plain Cloudflare-protected page: 1 credit for the page plus 10 for the bypass, so 11 credits per successful fetch. The free allowance of 1,000 credits therefore covers about 90 such pages, which is enough to validate a pipeline on real targets and not much more. At 10,000 pages a month you're looking at roughly 110,000 credits, and that number, not the headline plan size, is what you should be matching against a tier.

Run the arithmetic before you compare prices anywhere, including here. A job that mixes Amazon product pages (5 credits) with Cloudflare-protected blogs (11) and the occasional Google SERP (25) can land three times higher than a naive page count suggests. 👉 [Price your own URL list against the current tiers](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Which route fits your project

- **A one-off script, a few hundred pages, sites you can list by hand.** cfscrape or cloudscraper, plus Node 10+ if you go the cfscrape route. Accept the 5-second first hit and the UA/IP rules above.
- **A recurring job on a schedule, or a host where you can't install a JavaScript runtime.** The local solver becomes an operational dependency you patch yourself. This is where the hosted route starts paying for itself.
- **Sites serving reCAPTCHA rather than the standard interstitial.** cfscrape won't help. cloudscraper can, with a paid solver key on top. ScraperAPI bundles the bypass into the 10-credit surcharge.
- **Thousands of pages a month across mixed targets.** Compare your credit total, not the plan name, and decide whether auto-upgrade should be on.

If you're staying local, the two libraries are close enough that the deciding factor is your Python version: cfscrape's 2.6–3.7 window rules it out on anything modern, and cloudscraper's default native solver means no Node install. If you're moving to an API, the free 1,000 credits are the cheapest way to find out whether your targets actually pass.

## Before you pay

Confirm three things on your own URLs rather than on a sample: the per-URL credit cost from the Domain Cost Estimator for your ten hardest targets, whether your monthly volume leaves headroom above the tier you're eyeing, and whether you want auto-upgrade enabled, since it will move you up a plan automatically when credits run dry. If your job can't tolerate a pause at renewal, pay-as-you-go on Scaling and above is the alternative, capped at a monthly number you set.

Everything else is arithmetic you can do in a spreadsheet in ten minutes. 👉 [Check current ScraperAPI pricing and start with the free 1,000 credits](https://www.scraperapi.com/?fp_ref=coupons)
