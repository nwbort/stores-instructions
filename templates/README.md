# templates

Boilerplate that's the same in every `stores-*` repo. Copy, don't import —
these are starting points, and a repo is free to diverge once it has a reason
to.

| File | Goes to | What it is |
| --- | --- | --- |
| `scrape.yml` | `.github/workflows/scrape.yml` | The scheduled run: fetch, then commit only if changed. Knows nothing brand-specific; it just calls `./scrape.sh`. |
| `download.sh` | repo root | Fetch a URL, name the file after the URL, pick the extension from the MIME type, pretty-print JSON with `jq`. Sends browser-shaped headers, which is enough to get past most bot filtering. |
| `.gitignore` | repo root | Python leftovers. |
| `repo-README.md` | repo root, as `README.md` | README stub — fill in the brand and the source. |

`scrape.sh` is the one file you always write yourself. It's the entrypoint the
workflow calls, and everything brand-specific lives behind it. The simplest
possible version is one line:

```bash
#!/bin/bash
./download.sh 'https://api.example.com/StoreLocator/Stores'
```
