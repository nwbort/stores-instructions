# stores-instructions

How I build and run store-location scrapers.

Each brand I track gets its own repo (`stores-aldi`, `stores-auspost`,
`stores-ampol`, `stores-dan-murphys`, `stores-ww-new`, …). Every one of them
does the same thing: fetch a retailer's public store list on a schedule, write
it to JSON, and commit that JSON back to the repo when — and only when — it
actually changed. The git history is the point: it's a free, diffable record of
every store that opened, closed, moved or changed its hours.

This repo holds the principles behind that setup, so a new store repo can be
stood up without re-deriving them. It is deliberately not a framework — the
scraping code is different for every site and should stay that way.

## Read this

- **[PRINCIPLES.md](./PRINCIPLES.md)** — repo shape, output format, commit
  discipline, scheduling, secrets, and the failure modes worth designing
  against. This is the important one.
- **[SCRAPING.md](./SCRAPING.md)** — the ladder of approaches: try this first,
  then this, then this. General shapes only, not site specifics.
- **[templates/](./templates/)** — the boilerplate that's identical in every
  repo: `scrape.yml`, `download.sh`, `.gitignore`, README stub.

## Starting a new store repo

1. Create `stores-<brand>`, and set the **repo description to the source URL**
   (the workflow template bootstraps `scrape.sh` from it).
2. Copy in `templates/download.sh`, `templates/.gitignore`,
   `templates/repo-README.md` (as `README.md`) and `templates/scrape.yml` (to
   `.github/workflows/scrape.yml`).
3. Work down the ladder in [SCRAPING.md](./SCRAPING.md) until something
   returns the full store list.
4. Write `scrape.sh` as the single entrypoint that produces the JSON.
5. Pick a cron minute that no other store repo is using, and check that a
   second run in a row commits nothing.
6. Note in the repo's README where the data comes from and how to get it
   again if the source moves.
