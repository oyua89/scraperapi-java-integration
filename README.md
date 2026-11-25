# ScraperAPI Java: SDK 1.2 setup, why HtmlUnit proxy mode fails, and how far 5,000 free credits go

Three lines of Java are enough to get a request through ScraperAPI: add `com.scraperapi:sdk:1.2` to your build, construct `ScraperApiClient` with a key read from the environment, and call `.get(url).result()`. If you're on HtmlUnit, don't touch its proxy settings at all — point the browser at the API endpoint instead, because ScraperAPI authenticates with a key in the query string and HtmlUnit's proxy model has no clean way to send it.

The rest of this guide follows the order you'll hit problems in: which plan covers your workload, which of the three Java integration paths to use, what each request costs in credits, and where the timeouts and error codes will bite you.

> **Quick pick:** most Java projects should burn through the 5,000 free signup credits first, then take **Hobby at $49/month** (100,000 credits, 20 concurrent threads). Step up to **Startup at $149/month** only when rendered or premium requests push you past that allowance, since a single rendered premium page can cost 25 credits.
> 👉 [Compare the current ScraperAPI plans and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Which ScraperAPI plan does a Java project actually need?

Hobby covers a single scraper hitting plain HTML pages: 100,000 credits is 100,000 standard requests, and 20 concurrent threads is more parallelism than most scheduled jobs use. Startup exists for workloads that lean on JavaScript rendering, where the same 100,000 credits would only buy 10,000 pages.

Every plan includes the same feature set — JS rendering, premium residential and mobile IPs, anti-bot bypass, Autoparse, custom headers, CAPTCHA handling and rotating pools. What changes between tiers is the credit allowance, the concurrency cap, and geotargeting.

| Plan | Credits/month & threads | Monthly / annual (per month) | Who it fits | Get it |
| --- | --- | --- | --- | --- |
| Hobby | 100,000 · 20 threads | $49 / $44.10 | One scraper on static or lightly protected pages; US and EU geotargeting only | [Start on Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 · 50 threads | $149 / $134.10 | Render-heavy jobs, or several services sharing one key | [Start on Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 · 100 threads | $299 / $269.10 | Country-level geotargeting outside the US and EU | [Start on Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 · 200 threads | $475 / $427.50 | High volume plus pay-as-you-go overage | [Start on Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 · 300 threads | $975 / $877.50 | Sustained production load with priority support | [Start on Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 · 500 threads | $1,975 / $1,777.50 | Largest self-serve tier, priority routing | [Start on Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ · 500+ threads | Custom | Committed volume, dedicated team and Slack support | [Talk to sales about Enterprise](https://www.scraperapi.com/?fp_ref=coupons) |

Annual billing takes exactly 10% off across every self-serve tier. Two details that catch people out: credits reset at each renewal date and unused ones are lost, not carried forward; and pay-as-you-go overage only exists from Scaling upward. On Hobby, Startup and Business, running out of credits returns a 429 rather than an extra charge, so a batch job that suddenly starts failing every request is usually a billing signal, not a block.

Geotargeting is the other quiet upgrade trigger. Hobby and Startup are limited to US and EU proxy locations. If your code calls `.countryCode("jp")`, you need Business or above — and `countryCode` itself adds no credits, so the jump in plan price is entirely about location coverage.

## Three ways to call ScraperAPI from Java

| Path | Use it when | Limits |
| --- | --- | --- |
| Official SDK (`com.scraperapi:sdk:1.2`) | You want the fluent builder and typed parameters | Java 8+, Maven or Gradle, adds a dependency |
| API endpoint with query string | You're inside HtmlUnit, or want zero dependencies | You build and URL-encode the query yourself |
| Standard Java proxy + `Authenticator` | Existing `HttpURLConnection` code you don't want to rewrite | HtmlUnit cannot use this path |

### SDK: the shortest working path

Maven:

xml
<dependency>
  <groupId>com.scraperapi</groupId>
  <artifactId>sdk</artifactId>
  <version>1.2</version>
</dependency>


Gradle: `implementation 'com.scraperapi:sdk:1.2'`. Requires Java 8 or newer.

java
ScraperApiClient client = new ScraperApiClient(System.getenv("SCRAPERAPI_API_KEY"));
String html = client.get("https://example.com/").result();


Parameters chain in front of `.result()`:

java
String html = client.get("https://example.com/product")
    .render(true)
    .countryCode("gb")
    .sessionNumber(42)
    .result();


The key comes from the `SCRAPERAPI_API_KEY` environment variable every time — hardcoding it is the one mistake that reliably ends up in a public repo. Older tutorials show `com.scraperapi:sdk:1.0`; use 1.2, which is what the current Maven artifact and SDK reference document.

### Endpoint: build the URL yourself

java
String apiUrl = String.format(
    "https://api.scraperapi.com?api_key=%s&url=%s&render=true",
    apiKey,
    URLEncoder.encode(targetUrl, "UTF-8"));


Encoding the target URL is mandatory. Skip `URLEncoder.encode` and any target with its own query string — `?page=2&sort=price` is the common case — gets parsed as ScraperAPI's parameters instead of the site's.

### Proxy: works with plain `HttpURLConnection`, not with HtmlUnit

java
Authenticator.setDefault(new Authenticator() {
  protected PasswordAuthentication getPasswordAuthentication() {
    if (getRequestorType().equals(RequestorType.PROXY))
      return new PasswordAuthentication("scraperapi", apiKey.toCharArray());
    return null;
  }
});
System.setProperty("http.proxyHost", "proxy-server.scraperapi.com");
System.setProperty("http.proxyPort", "8001");
System.setProperty("jdk.http.auth.tunneling.disabledSchemes", "");


The last line matters: without it, the JDK refuses to send proxy credentials over an HTTPS tunnel and you get an authentication failure that looks like a bad key. Request parameters still travel as headers here, e.g. `connection.setRequestProperty("x-sapi-render", "true")`. Credentials themselves must go through the proxy authenticator — putting the API key only in `x-sapi-api_key` while using proxy mode will not authenticate you.

## HtmlUnit: use the endpoint, skip the proxy

HtmlUnit's proxy configuration can be set, but it expects either a static IP or basic authentication against a host you name. ScraperAPI's key lives in the URL query string and its proxy endpoint routes by parameter, so the two models don't line up. Pointing HtmlUnit's proxy at `proxy-server.scraperapi.com` produces connection and auth errors rather than responses.

The working pattern is to hand HtmlUnit the ScraperAPI endpoint as if it were the target page:

java
String apiUrl = String.format("https://api.scraperapi.com?api_key=%s&url=%s",
    apiKey, URLEncoder.encode("https://quotes.toscrape.com", "UTF-8"));

WebClient webClient = new WebClient(BrowserVersion.CHROME);
webClient.getOptions().setCssEnabled(false);
webClient.getOptions().setJavaScriptEnabled(false);
webClient.getOptions().setThrowExceptionOnFailingStatusCode(false);
webClient.getOptions().setThrowExceptionOnScriptError(false);

HtmlPage page = (HtmlPage) webClient.getPage(apiUrl);
DomNodeList<DomNode> quotes = page.querySelectorAll(".quote");


With `htmlunit:2.70.0` and `java-dotenv:5.2.2` on the classpath, `mvn compile exec:java -Dexec.mainClass=MarketPrice` runs it. Keep JS and CSS disabled in HtmlUnit even when you send `render=true` upstream: ScraperAPI's headless Chrome does the rendering, and HtmlUnit re-rendering the returned HTML only slows the parse down.

## What each request costs in credits

| Configuration | Credits per request |
| --- | --- |
| Standard `.result()` | 1 |
| `.render(true)` | 10 |
| `.premium(true)` | 10 |
| `.premium(true).render(true)` | 25 |
| `.ultraPremium(true)` | 30 |
| `.ultraPremium(true).render(true)` | 75 |

`countryCode`, `sessionNumber`, `deviceType` and `autoparse` add nothing. The domain you're scraping changes the base rate though: Amazon, Walmart and eBay pages cost 5 credits, Google and Bing cost 25, LinkedIn costs 30, and sites behind Cloudflare, DataDome or PerimeterX add 10 when bypass logic kicks in.

Two rules save real money. First, `premium` and `ultraPremium` are mutually exclusive — chaining both is invalid, and the escalation ladder runs standard (1) → render (10) → premium (10) → premium + render (25) → ultraPremium (30) → ultraPremium + render (75). Second, don't enable `.render(true)` by default. Try the plain request first; a rendered page costs ten times as much and adds latency, and static HTML comes back identical either way. `render=true` is available on every plan, including the free credits, so there's no upgrade gate on testing it.

If you only need one rendered page a day, this table is academic. If you're rendering 5,000 product pages a month, it's the difference between Hobby and Startup.

👉 [Check which plan covers your monthly credit burn](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## What 5,000 free credits actually buys

Signing up gives you 5,000 credits without a credit card. How far that goes depends entirely on which parameters you send:

| Request type | Pages from 5,000 credits |
| --- | --- |
| Standard page | 5,000 |
| `.render(true)` | 500 |
| `.premium(true).render(true)` | 200 |
| `.ultraPremium(true).render(true)` | ~66 |
| Google search (25 credits each) | 200 |

That last row is why "5,000 free credits" reads differently to different people. A developer validating a scraper against plain HTML gets thousands of test runs. Someone testing against Google or a Cloudflare-protected site gets a few hundred at best. Budget your trial accordingly, and check the Domain Cost Estimator in the dashboard before pointing a loop at a new site.

After the signup credits, the free tier is 1,000 credits a month — enough to keep a small scheduled job alive, not enough to run anything real.

## Error codes, retries, and the timeout that fakes failures

| Code | Meaning | Fix |
| --- | --- | --- |
| 200 | Success | — |
| 401 | Invalid API key | Check `SCRAPERAPI_API_KEY` is actually set in the process |
| 403 | Blocked, or credits exhausted | Escalate to premium/ultraPremium, or top up |
| 404 | Target page doesn't exist | Fix the URL, not the proxy |
| 429 | Rate limit, or credits gone on Hobby–Business | Lower concurrency, or move to a plan with pay-as-you-go |
| 500 / 503 | Transient error, **not charged** | Back off and retry |

500 and 503 responses don't consume credits, so retrying them costs nothing but time. That matters because ScraperAPI retries internally for up to 70 seconds before giving up. A client-side timeout set to 5 seconds will kill requests that were about to succeed, and you'll see a pile of phantom failures that never appear in your dashboard. Set socket and connect timeouts well above 70 seconds and let `.retry(n)` handle the rest — the SDK defaults to 3 attempts on non-200 responses, and raising it just multiplies your worst-case wait.

Sessions expire 15 minutes after their last use. Any integer works as an ID, so paginated or login-gated flows need the whole sequence inside that window, or a fresh `sessionNumber` and a fresh login.

## Batch jobs and structured endpoints

Anything above roughly 20 URLs shouldn't go through `.result()` in a loop — it blocks a thread per request and makes concurrency limits your bottleneck. Post to `https://async.scraperapi.com/jobs` with `{"apiKey": "...", "url": "..."}`, take the `statusUrl` from the response, and poll it every 5 seconds until the status is `finished` (read `response.body`) or `failed`.

For supported sites, skip parsing entirely. `https://api.scraperapi.com/structured/google/search?api_key=...&query=...`, `structured/amazon/product` with an `asin`, and `structured/walmart/search` with `query` and `tld` return JSON instead of HTML. That also sidesteps the per-page HTML parsing you'd otherwise maintain whenever the target site redesigns.

## FAQ

**Can HtmlUnit use ScraperAPI's proxy mode?** No. ScraperAPI authenticates via URL query string; HtmlUnit's proxy layer expects a static IP or basic auth. Use the API endpoint method instead.

**Which SDK version should I use?** `com.scraperapi:sdk:1.2`. Some older documentation still shows 1.0.

**When should I turn on `render=true`?** When the page is a React/Vue/Angular SPA, or when a first scrape returns an empty shell. Not by default — it costs 10 credits against 1.

**How long does a session last?** 15 minutes after its last request. Paginated flows need to finish inside that window.

**Do failed requests cost credits?** 500 and 503 don't. Check your dashboard before assuming a retry loop is free across other status codes.

## How to choose

Match the parameter to the problem, not the plan tier to your ambition. Static pages that load fine → standard requests, 1 credit each, Hobby is enough. Content only appears after JavaScript runs → `.render(true)`, and recalculate your monthly volume at 10 credits a page before picking a tier. Datacenter IPs get blocked → `.premium(true)`. Cloudflare or DataDome with everything thrown at it → `.ultraPremium(true)`, at 30 to 75 credits a page. More than 20 URLs per run → the async endpoint, whatever plan you're on.

Before you pay, confirm two things on the pricing page: whether your targets sit outside US/EU (that's the Business threshold) and whether your monthly credit estimate assumes rendering, because a 10× multiplier turns a comfortable allowance into a two-week one. Credits don't roll over, so size for a normal month, not a peak one.

👉 [Start with 5,000 free credits, no card required](https://www.scraperapi.com/?fp_ref=coupons)
