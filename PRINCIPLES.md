# Principles

The rules of thumb behind every `stores-*` repo. Deviate when a site forces it,
but deviate knowingly.

## 1. The repo is the dataset

The committed JSON *is* the deliverable. There's no database, no API, no
external storage — a store repo is a flat directory with a scraper, a workflow,
and the data it produces.

That means git history is the changelog. Every commit answers "what changed
about this brand's store network today", and `git log -p stores.json` is the
whole product. Protect that property; most of the rules below exist to keep the
history clean enough to be worth reading.

**One brand per repo.** Don't build a monorepo of scrapers. Sites break
independently, run on different cadences, and have wildly different shapes — a
broken Ampol scraper should never stop the Woolworths data from updating.

**Say where the data comes from in the README.** The source URL, and whatever
you needed to find it — which page, which request, which header. It's the first
thing you want when you come back in six months and the locator has moved.

## 2. Output: JSON, boringly formatted, stably ordered

- **`stores.json` at the repo root** is the default name. New repos should use
  it.
- **Don't rename an existing output file.** Several repos carry URL-derived
  names from how they started (`store.aldi.com.au-stores.json`,
  `api.danmurphys.com.au-apis-ui-StoreLocator-Stores-danmurphys.json`). Ugly,
  but renaming severs the file's history, which is the thing of value. Keep the
  historical filename even when the scraping method behind it changes
  completely.
- **Pretty-print, 2-space indent, `ensure_ascii=False`.** One field per line.
  Minified JSON produces a single-line diff on every change, which throws away
  the entire benefit of committing it.
- **Sort by something stable** before writing — an id, a ref, or a
  `(state, suburb, name)` tuple. Upstream APIs return results in arbitrary and
  unstable order; without a sort, every run looks like the whole file changed.
  This applies to nested lists too (services, opening hours) — sort them, or map
  them to a fixed day order.
- **Normalise when upstream is ugly, pass through when it's clean.** Dan
  Murphy's returns one tidy JSON document, so it's saved almost verbatim.
  AusPost's radius API returns something unusable, so it gets flattened into a
  designed schema. Both are fine. Don't do transformation work for its own sake.
- **Commit the generated JSON and nothing else.** Intermediates — downloaded
  sitemaps, locator landing pages, raw API dumps — are working files. Write them
  somewhere ignored, or to a temp directory, and let them be thrown away. Some
  of the older repos do commit their raw sitemap or HTML; that's history, not a
  pattern to copy. The store list is what's being tracked, and every extra file
  in the repo is noise in the diff that made you look.

## 3. Only commit when the data actually changed

A daily job that commits unconditionally turns the history into noise and makes
"when did this store close?" unanswerable.

The cheap version, which is enough for most repos — let git decide, and exit
cleanly when there's nothing to commit:

```bash
git add stores.json
git commit -m "Update store data: $(date -u)" || exit 0
git pull --rebase
git push
```

Name the output files explicitly rather than `git add -A`, so a stray
intermediate can never sneak into a commit (see above).

**The trap: volatile fields.** If the output embeds a `generated_at` timestamp,
a request id, or a server-side "last updated", the file differs on *every* run
and the guard above never fires. This bit `stores-auspost`.

Two ways out, in order of preference:

1. **Don't put volatile fields in the committed file.** Run metadata belongs in
   the Actions log and the commit timestamp, not in the data. This is the right
   answer for new repos.
2. **Compare with the volatile fields stripped**, when metadata genuinely earns
   its place in the file:

   ```bash
   if diff -q \
     <(git show HEAD:stores.json | jq 'del(.generated_at)') \
     <(jq 'del(.generated_at)' stores.json) >/dev/null; then
     echo "No changes besides generated_at"
     exit 0
   fi
   ```

Always `git pull --rebase` before pushing — a manual dispatch racing the
scheduled run shouldn't fail the job.

## 4. Scheduling

- **Daily is the right default.** Store networks change slowly. Hourly is
  almost never justified and burns goodwill with the source.
- **Pick an odd minute, off the hour.** `'23 6 * * *'`, `'33 7 * * *'`,
  `'17 3 * * *'`. Everyone's cron fires at `:00`, and GitHub queues those runs
  for many minutes.
- **Stagger repos against each other** so a bad morning doesn't mean five
  simultaneous failures to untangle.
- **Always include `workflow_dispatch`.** You will want to trigger a run by
  hand, constantly.
- **`on: push` is useful on small repos** for fast iteration, but drop it if
  the scrape is slow or the source is rate-sensitive.
- **Tier the cadence when the data has tiers.** `stores-ww-new` refreshes the
  cheap store *index* from a sitemap daily, and the expensive per-store
  *details* (one request per store) weekly. Don't hammer a site nightly for
  data that changes quarterly.
- Boilerplate worth keeping: `permissions: contents: write`,
  `if: ${{ !github.event.repository.is_template }}`, and — for long jobs — a
  `concurrency` group plus `timeout-minutes` so a hung run can't stack up.

## 5. Secrets and public API keys

Store locators often need a key that's shipped in the site's own frontend
JavaScript — visible to anyone with devtools open. It isn't really a secret, but
it still shouldn't be hardcoded:

- It goes **stale** (AusPost rotates theirs), and a hardcoded value means a code
  change to fix a config problem.
- Secret scanners flag it regardless of how public it actually is.

So: read it from an env var, wire it up as a repo secret, and **fail fast with
an actionable message** if it's missing — the error should say where to get a
fresh value, not just `KeyError`. Then write down the retrieval steps in the
repo README (which page, which request, which header). Future-you re-does this
at some point; make it a five-minute job.

## 6. Design against silent failure

The worst outcome isn't a red X — it's a green tick over empty data.
`stores-aldi` cheerfully committed `"total_stores": 0` for every state after
ALDI migrated their locator to a SPA. The job passed. Nobody noticed.

- **Never write an empty result quietly.** At minimum print a loud warning; at
  best, exit non-zero when the store count is zero.
- **Sanity-check the magnitude.** A drop from 5,700 stores to 40 is a broken
  selector, not a mass closure. A threshold check (say, ">50% drop fails") is
  cheap insurance.
- **Fail on the fetch, warn on the parse.** A network error should stop the run;
  a single unparseable store out of 2,000 shouldn't.
- **Retry with exponential backoff and jitter** on 429 and 5xx, and give up
  after a bounded number of attempts.
- **Cap concurrency.** ~8 workers is plenty when fetching thousands of detail
  pages. Parallelism here is about finishing in ten minutes instead of two
  hours, not about going as fast as possible.
- **Send realistic browser headers** (User-Agent, Accept, Accept-Language,
  `--compressed`) when a site rejects bare `curl`. This is about looking like a
  normal client, not about evading anything.
- `set -e` in every shell script.

## 7. Scope and manners

Only public store-locator data — the same information the site shows any
visitor: name, address, coordinates, phone, trading hours, services. Nothing
behind a login, no personal data, no pricing or stock scraping under cover of a
store scrape.

Run daily, back off when asked to, keep concurrency modest, and prefer the
endpoint the site itself uses over anything clever.

## 8. Tooling: keep it small

- **Bash + `curl` + `jq`** when the source is a single request that needs no
  reshaping. Dan Murphy's whole scraper is two lines.
- **Python standard library** (`urllib`, `json`, `xml.etree`) for anything that
  needs parsing. No `requirements.txt` to maintain, no install step, no
  dependency drift. Reach for `requests` only when retries and sessions make it
  genuinely worth the file.
- **`scrape.sh` is the entrypoint.** The workflow calls `./scrape.sh` and
  nothing else, so the workflow stays identical across every repo and all the
  brand-specific mess lives in one place.
- **One script per job**, composed by `scrape.sh`: `download.sh` fetches,
  `parse.sh` transforms. Small pieces you can run by hand while debugging.
- **What CI runs must run locally**, with the same command. If reproducing a
  failure requires pushing to a branch, the scraper is too clever.
- **Comment the *why*, especially after a break.** The header of
  `stores-aldi/scrape_stores.py` explains that the site moved from a Yext
  directory to a Nuxt/Uberall SPA and that the old class selectors are gone.
  That paragraph is worth more than the code beneath it.
