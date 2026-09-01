# stores-<brand>

Scrapes <brand>'s public store locator into [`stores.json`](./stores.json),
refreshed daily by a GitHub Actions run that commits only when the data
actually changed.

## Source

<source URL> — <one line on what it is: sitemap / locator API / vendor
endpoint, and anything needed to get at it again.>

## Running locally

```bash
./scrape.sh
```

## Conventions

See [stores-instructions](https://github.com/nwbort/stores-instructions).
