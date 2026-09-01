# Finding the data

How the scraping happens is different for every site, and that's fine. What's
consistent is the *order you try things in*. Work down this ladder and stop at
the first rung that gives you the whole store list — each step down costs more
to build and breaks more often.

Before anything else: open the store locator in a browser with devtools on the
Network tab, search for a suburb, and watch what the page actually requests.
Nine times out of ten the answer is sitting right there.

## 1. Sitemaps

Try `/sitemap.xml` first, then any locator-specific one — Woolworths publishes
`/sitemap-store-locator.xml`, Ampol's main sitemap lists every store page.

Cheap, stable, cache-friendly, and it gives you a complete list of stores for
one request. Often the slug alone carries an id, a name and a state
(`/storelocator/nsw-neutral-bay-1234`), which may be all you need. If not,
it's a perfect index for step 7.

## 2. The site's own JSON API

Most locators are a thin frontend over an internal JSON endpoint. In the best
case one unauthenticated request returns every store — Dan Murphy's
`api.danmurphys.com.au/apis/ui/StoreLocator/Stores/danmurphys` is the whole
scraper.

Look for paths containing `StoreLocator`, `locations`, `stores`, `find`. Check
whether the parameters can be widened: a `radius=25` in the UI often accepts
much more, and a `pageSize` cap is worth probing. If the endpoint needs a key
from the page's JavaScript, see the secrets section in
[PRINCIPLES.md](./PRINCIPLES.md#5-secrets-and-public-api-keys).

## 3. Third-party locator platforms

Plenty of brands outsource their locator. Recognise the vendor by the hostname
in the network trace — **Uberall**, **Yext**, **Stockist**, **Algolia**,
**Bullseye**, **PlaceMakers**, **Brandify**, **StoreRocket**. ALDI's locator is
Uberall, which exposes a single `storefinders/<key>/locations/all` endpoint that
returns the national list in one hit.

These are worth recognising because each vendor has a documented, stable API
shape, usually including an "all locations" call the brand's own UI never makes.
Once you've identified the vendor, the work is mostly done.

## 4. Structured data embedded in the page

If there's no API, look inside the HTML *before* you look at the HTML:

- **JSON-LD** — `<script type="application/ld+json">` with a `LocalBusiness`
  object. This is how Ampol's per-store pages are scraped: name, address, geo,
  `openingHoursSpecification` and services all come out of one clean blob.
  Handle both a bare object and an `@graph` array.
- **Framework state** — `__NEXT_DATA__` (Next.js), `window.__NUXT__` (Nuxt),
  or an inline `<script>` assigning a store array. Modern SPA locators almost
  always ship their data this way.

Both are far more stable than markup, because they're contracts with search
engines and hydration code rather than presentational classes.

## 5. Parsing HTML

Last resort. Class names and DOM structure change without warning and without
any error — you just start getting zero results. See the ALDI story in
[PRINCIPLES.md](./PRINCIPLES.md#6-design-against-silent-failure).

If you must: anchor on the most semantically meaningful thing available (an
`href` pattern, a `data-` attribute, an `itemprop`) rather than a styling class,
and make an empty result loud.

## 6. Grid / radius search

Some APIs only answer "what's near this point" and have no way to list
everything. AusPost is the case in point. The workaround:

1. Lay a grid of points over the country — spacing must be less than
   `radius × √2` so the circles overlap with no gaps. ~300 km spacing with a
   250 km radius covers Australia in around 340 points.
2. Paginate each point (`offset`/`size`) until exhausted; caps are usually
   ~100 per page.
3. **Dedupe by store id** across overlapping circles.
4. Remember the outlying bits — Norfolk, Lord Howe, Christmas, Cocos — if the
   brand serves them.

Expensive (AusPost takes ~7 minutes) but completely reliable. Add a small
inter-request delay and set `timeout-minutes` on the job.

## 7. Two-phase: index then details

When the cheap list doesn't carry enough fields, split it in two, as
`stores-ww-new` does: a sitemap gives the store index daily, and a separate
per-store API call fills in details. Fetch details with bounded concurrency
(~8 workers), and run that phase on a slower cadence — weekly is usually right,
since addresses and trading hours change far more slowly than the store list.

Write each store's detail to its own file (`store-details/<id>.json`) rather
than one giant blob: the diffs stay tiny and readable, and a single failed
fetch doesn't poison the whole file.

## 8. Headless browser

Playwright, or `shot-scraper` for the simple cases. Only when the data is
genuinely constructed at runtime and steps 1–7 all fail.

It's slow, needs a browser cache step in CI, and adds real dependencies — so
treat reaching for it as a signal to go back and look harder at the network
trace first. If you do use it, still try to grab the underlying XHR response
rather than scraping the rendered DOM.

## When it breaks

It will. Sites get rebuilt and scrapers go quiet rather than loud.

1. Check the committed raw artefact (sitemap, landing page) — the diff usually
   shows the migration.
2. Re-run the devtools pass from the top of this ladder. The answer is often
   that the site moved *up* a rung: a hand-rolled HTML directory became a SPA
   with a clean vendor API behind it, which is a better scraper than the one
   you had.
3. Keep the output filename and schema stable even when the method changes
   completely, so the history stays continuous.
4. Write down what moved and why, in a docstring at the top of the scraper.
