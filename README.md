# Hi Thuc — hithuc.com

Static site rebuilt **1:1 from the Framer site**, with every asset self-hosted.
No framework — Cloudflare serves the files in `./public` from the edge, fronted by
a tiny Worker (`worker/index.js`) that only exists to handle Framer's CMS range
protocol (see below). All 15 pages (home, `/work` + 4 projects, `/blog` + 8 posts)
are included as full pre-rendered HTML.

## Why there's a Worker

Framer's client-side router (used when you click between pages without a full
reload) loads collection/detail content by fetching byte ranges of the CMS data
files (`public/framer/cms/**/*.framercms`), encoding the range in the query string
as `?range=<from>-<to>` (inclusive). Framer's CDN honours that query; a plain
static host ignores it and returns the whole file, so the CMS reader gets the wrong
bytes and the page renders **blank after client-side navigation** (direct loads are
fine — they're pre-rendered). `worker/index.js` reads the file and returns the exact
requested slice. Everything else passes straight through to static assets.

## Layout

```
public/
  index.html              # home (from the site2code export)
  work/index.html         # projects list
  work/<slug>/index.html  # 4 project detail pages
  blog/index.html         # blog list
  blog/<slug>/index.html  # 8 blog post pages
  framer/
    assets/               # Inter fonts (woff2) + Open Graph / share images
    images/               # all page images (originals; serve every srcset size)
    sites/74mJ.../        # Framer JS runtime + React/Motion bundles + search index
    modules/              # CMS loader modules + .framercms data (initial render)
    cms/                  # same .framercms data, served with ?range= slicing (SPA nav)
    third-party-assets/   # Archivo font (fontshare)
worker/index.js          # serves *.framercms ?range= slices; delegates the rest
wrangler.jsonc            # Worker + static-asset config + custom domain
```

Every `https://framerusercontent.com/...` reference in the HTML **and** inside the
JS/JSON bundles was rewritten to a local `/framer/...` path, so the site loads with
no calls to Framer's CDN for static assets. The Framer bundle resolves CMS data at
runtime via `new URL("./x.framercms", base)`, so in JS the host is rewritten to the
**absolute** `` `${location.origin}/framer` `` (a relative base would throw).

## Develop

```bash
npm install
npm run dev          # wrangler dev — serves ./public locally
```

## Deploy

```bash
npm run deploy       # wrangler deploy → hithuc.com + www.hithuc.com
```

## Updating from the Framer site

Re-run `node scripts/mirror.mjs`. It reads the home page from the local export
(`SRC`), discovers every other route from the live site's `sitemap.xml`, fetches
each one as full pre-rendered HTML (using a crawler User-Agent — Framer serves a
lightweight client-render shell to normal browsers and warms up SSR lazily, so the
script retries until it gets the full page), re-downloads all assets into
`public/framer/`, and writes each page to `public/<route>/index.html`.
Update `LIVE`/`SRC`/`PROD` at the top of the script if the source URLs change.

## Notes / known limits

- **Self-contained for rendering.** Fonts, images, styles, and the layout JS are
  all local — the page renders fully without Framer's CDN.
- **Dynamic Framer features still call home.** Site search, form submissions, and
  Framer analytics talk to `api.framer.com` at runtime; these are not mirrored and
  will no-op/fail gracefully on a static host.
- **Voice widget is third-party.** The Sellvoxa chat widget loads from
  `sellvoxa.com/widget.js` (live service — intentionally left external).
- **Page titles.** Collection/detail pages keep the Framer template's `- Majd`
  site-name suffix (that's how the source site is published); only the home title
  was customised to `Hi Thuc`.
