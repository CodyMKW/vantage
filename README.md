# Vantage

A sleek game storefront that compares live PC game prices across Steam, GOG, Humble, Fanatical, Epic and more, plus a Nintendo section with live eShop prices. No build step, no framework: the whole site is one `index.html`, with an optional tiny Cloudflare Worker (in `worker/`) for Nintendo prices.

Built by **CodyMKW** and **Joseph** as a vibe-coded take on the Epic Games Store layout, with better game listings.

> Vantage is an independent fan project. It is not affiliated with Epic Games, Valve, or any storefront. Purchases always complete on the store's own site.

---

## Features

- **Discover page**: auto-rotating hero banner with a side list of featured picks, plus scrollable rows: *Free to claim right now*, *Free on PC stores*, *Top deals*, *Top rated on Nintendo Switch* (when a RAWG key is set), *Critically acclaimed*, *Great games under $10*, *Deepest discounts*
- **Portrait game cards** with discount badges, strike-through original price, and a wishlist heart
- **Browse page** with filters: sort, store, max price, minimum Metacritic, on-sale only, AAA only, and Load more pagination
- **Live search** with suggestions as you type (arrow keys + Enter supported)
- **Game detail view**: hero art, Metacritic and Steam ratings, release date, publisher, price comparison across every store, lowest price ever, and links to Steam and Metacritic
- **Nintendo section**: the Switch (and Switch 2) catalog with Popular / Top rated / New releases / Coming soon tabs, search, ratings, screenshots, and store links (including the Nintendo eShop). A **Grid / List** toggle gives a dense, table-style view
- **Nintendo eShop prices** (needs the optional price Worker, see [below](#nintendo-prices-the-price-worker)): current price, sale price and `-NN%` badge on every Nintendo card and row, plus United States / Canada / Mexico prices and the sale end date on each game page
- **Free games page**: live giveaways filtered by platform (PC, Steam, Epic, GOG, Switch, PlayStation, Xbox, Android, iOS) and type (games, DLC & loot, early access)
- **Wishlist** that refreshes prices on open and flags **price drops** (Nintendo games too, once the price Worker is set up)
- **Cart** with totals and savings; checkout opens each game's best deal in the store
- Shareable URLs for every game (`#/game/<id>`)
- Skeleton loading states, retry buttons on errors, responsive layout down to phones, reduced-motion support

## Quick start

No install. Just open the file:

```bash
git clone https://github.com/<you>/vantage.git
cd vantage
# open index.html in a browser, or serve it locally:
python3 -m http.server 8080
# then visit http://localhost:8080
```

Opening via `file://` works for most things, but serving over HTTP is recommended.

## Deploying

### GitHub Pages
1. Push `index.html` to the repo root.
2. **Settings → Pages → Build and deployment** → Source: *Deploy from a branch* → `main` / `/ (root)`.
3. Visit `https://<you>.github.io/<repo>/`.

### Nekoweb
Upload `index.html` to your site's root through the Nekoweb dashboard (or its API/CLI). It's fully static, so nothing else is needed.

### Cloudflare Pages / any static host
Point it at the repo with no build command and `/` as the output directory.

## File structure

```
vantage/
├── index.html          # the entire site: HTML + CSS + JS
├── README.md
└── worker/             # optional: Nintendo price service (Cloudflare Worker)
    ├── index.js        # the Worker (no dependencies, no build step)
    └── wrangler.toml   # Worker name, compatibility date, config vars
```

Deploy `index.html` to your static host as before; deploy `worker/` separately to Cloudflare (see the Worker section). The site works without the Worker, it just won't show Nintendo prices.

Inside `index.html`, in order:

| Section | What's in it |
|---|---|
| `<style>` | Design tokens (CSS variables), layout, cards, modal, drawer, responsive rules |
| `<body>` | Header, five views (`#home`, `#browse`, `#nintendo`, `#free`, `#wishlist`), footer, modal, cart drawer, toasts |
| `<script>` | API helpers, home rows + hero, browse filters, wishlist, game detail, cart, search, router |

## Data sources

Vantage combines these sources:

| Source | Used for | Key? |
|---|---|---|
| **[CheapShark](https://apidocs.cheapshark.com/)** | PC prices, deals, store info, ratings | No |
| **Steam CDN** | PC artwork: `https://cdn.cloudflare.steamstatic.com/steam/apps/<appid>/…` | No |
| **[RAWG](https://rawg.io/apidocs)** | Nintendo Switch catalog: art, ratings, release dates, screenshots, store links | **Yes, free** |
| **[GamerPower](https://www.gamerpower.com/api-read)** | Live free-game giveaways | No |
| **Nintendo eShop price API** (via the price Worker) | Nintendo prices for US / CA / MX | No (unofficial) |
| **Nintendo catalog search** (via the price Worker) | Matching a game title to its Nintendo ID (`nsuid`) | No (public search key, unofficial) |

### Setting up RAWG (Nintendo section)

RAWG needs a free API key. Without one, the whole site still works; the Nintendo page just shows a setup card.

1. Create a free account at [rawg.io/apidocs](https://rawg.io/apidocs) and copy your key.
2. Either set it in `index.html` so it works for every visitor:
   ```js
   const RAWG_KEY='your-key-here';
   ```
   or leave it empty and paste the key into the setup card on the Nintendo page (saved in that browser only).

RAWG's free tier allows 20,000 requests/month for non-commercial projects and requires a visible credit link on pages that show its data (Vantage includes this in the footer, on the Nintendo page, and in the game modal). RAWG data can't be redistributed. Since this is a static site, the key is visible in the page source; that's normal for RAWG browser usage, but don't reuse a key you care about elsewhere.

### Nintendo prices

There's no official, documented, free Nintendo eShop price API, so Nintendo prices come from the optional price Worker (full docs in [Nintendo prices (the price Worker)](#nintendo-prices-the-price-worker)). Without it, the Nintendo section shows ratings and store links but **no prices**, and each game page sends you to the eShop.

Art fallback chain: `library_600x900.jpg` (cards) / `library_hero.jpg` (banners) → `header.jpg` → CheapShark thumbnail → text placeholder.

### Endpoints used

Base URL: `https://www.cheapshark.com/api/1.0`

| Endpoint | Used for |
|---|---|
| `/deals?...` | Home rows, hero, and Browse results |
| `/deals?id=<dealID>` | Extra game info on the detail view (ratings, publisher, release date) |
| `/games?title=<q>&limit=6` | Search suggestions |
| `/games?id=<gameID>` | Per-store price comparison and lowest price ever |
| `/games?ids=<id,id,…>` | Wishlist price refresh (batched in groups of 25) |
| `/stores` | Store names and logos (cached 24h) |
| `/redirect?dealID=<dealID>` (site root) | Outbound "Buy" links for non-Steam stores |

**Buy links:** `dealID`s from the API are already URL-encoded (for example `abc%2Bdef%3D`), so Vantage decodes then re-encodes them once; encoding them twice makes `/redirect` fail and dump you on the CheapShark home page. Steam deals skip CheapShark entirely and link straight to `https://store.steampowered.com/app/<appid>`. Every other store goes through CheapShark's redirect, which is how CheapShark earns a commission from some stores (per its [About page](https://www.cheapshark.com/about)); Vantage says so in the game modal and footer.

### `/deals` parameters Browse sends

| Param | Source |
|---|---|
| `pageSize` | `24` |
| `pageNumber` | increments on Load more |
| `sortBy` / `desc` | Sort dropdown (see `SORTS` in the script) |
| `title` | Title filter / search |
| `storeID` | Store dropdown |
| `upperPrice` | Price dropdown (`0` = free) |
| `metacritic` | Minimum Metacritic dropdown |
| `onSale` | "On sale only" checkbox |
| `AAA` | "AAA titles only" checkbox |

### RAWG endpoints used

Base URL: `https://api.rawg.io/api` (every request appends `key=<RAWG_KEY>`)

| Endpoint | Used for |
|---|---|
| `/platforms/lists/parents` | Finds the Nintendo Switch / Switch 2 platform IDs (falls back to `7`) |
| `/games?platforms=…&ordering=…&metacritic=…&dates=…&search=…&page_size=24&page=n` | Nintendo grid, tabs, search, and the home row |
| `/games/<id>` | Game page: description, developers, publishers, ESRB, ratings |
| `/games/<id>/stores` | "Where to buy" links |
| `/games/<id>/screenshots` | Screenshot strip |

Nintendo tabs map to RAWG ordering: Popular `-added`, Top rated `-metacritic` with `metacritic=80,100`, New releases `-released` over the last 90 days, Coming soon `released` over the next 400 days. Nintendo game IDs are namespaced `r<id>` inside Vantage so they never collide with CheapShark IDs.

### GamerPower endpoint used

`GET https://gamerpower.com/api/giveaways?platform=<p>&type=<game|loot|beta>&sort-by=popularity` (multiple platforms are joined with `.`, for example `ps4.ps5`). No key; please stay under 10 requests/second. GamerPower asks for attribution and doesn't allow reselling its data.

### Deal shape (what the app reads)

```jsonc
{
  "title": "…", "dealID": "…", "gameID": "…", "storeID": "1",
  "salePrice": "9.99", "normalPrice": "19.99", "savings": "50.025",
  "metacriticScore": "88", "steamRatingText": "Very Positive",
  "steamRatingPercent": "92", "steamRatingCount": "12345",
  "steamAppID": "12345", "releaseDate": 1600000000, "thumb": "https://…"
}
```

Each deal is normalized into `{ id, dealID, title, sale, normal, savings, mc, pct, rtext, rcount, app, release, store, thumb }` by `norm()`. Results are de-duplicated per game, keeping the cheapest store.

## Nintendo prices (the price Worker)

Nintendo has no official free price API, and a static page can't safely call Nintendo's endpoints directly (CORS, and you'd hit them on every page view). So the prices come from a tiny Cloudflare Worker in `worker/` that Vantage calls instead.

```
browser (Vantage) ──► Worker (worker/index.js) ──► api.ec.nintendo.com/v1/price   game prices
                         │  caches answers    ──► <app>-dsn.algolia.net            title → Nintendo ID (nsuid)
                         └─ adds CORS headers
```

**Why not DekuDeals?** It has no public API, and scraping it would mean hitting their servers and republishing the price data they built (browsers would block it anyway). Vantage uses Nintendo's own price endpoint instead. That means **no price history**: Nintendo only reports today's price and any active sale.

### Setup

The Worker is one file with no dependencies, so there are three ways to deploy it. Pick one.

**Option A: Cloudflare dashboard (no Node.js needed)**

1. In the Cloudflare dashboard go to **Workers & Pages → Create → Create Worker**, name it `vantage-prices`, and click **Deploy** (the starter "Hello World" code is fine for now).
2. Click **Edit code**, delete everything in the editor, paste in the full contents of `worker/index.js`, and click **Deploy**.
3. Optional: under the Worker's **Settings → Variables and secrets**, add `ALLOWED_ORIGIN` (see [Config](#config-workerwranglertoml)).

Menu labels move around occasionally, but the flow is always: create a Worker, paste the code, deploy. To update later, paste the new `worker/index.js` over the old code and deploy again.

**Option B: Wrangler CLI with Node.js 22+** (current Wrangler requires Node 22 or newer; check with `node -v`)

```bash
cd worker
npx wrangler login      # once; opens a browser to authorize your Cloudflare account
npx wrangler deploy     # publishes to https://vantage-prices.<your-subdomain>.workers.dev
```

**Option C: Wrangler CLI on Node.js 18 or 20** (if you see `Wrangler requires at least Node.js v22.0.0`)

Pin Wrangler 3, which supports Node 16.17 and newer. Use `npx wrangler@3` wherever this README says `npx wrangler`:

```bash
cd worker
npx wrangler@3 login
npx wrangler@3 deploy
```

It prints an "out-of-date" warning; that's safe to ignore for this single-file Worker. Or upgrade Node with [nvm](https://github.com/nvm-sh/nvm) / [nvm-windows](https://github.com/coreybutler/nvm-windows) / [Volta](https://volta.sh/) and use Option B.

After deploying, with any option:

1. Open `https://vantage-prices.<your-subdomain>.workers.dev/status`. You want `"ok": true`. If it's `false`, the response says which upstream failed (see [Troubleshooting](#troubleshooting-the-worker)).
2. In `index.html`, set the Worker URL with **no trailing slash**:
   ```js
   const PRICE_API='https://vantage-prices.<your-subdomain>.workers.dev';
   ```
3. Reload the Nintendo page. Cards show prices; open a game to see US / Canada / Mexico.

Local testing: `npx wrangler dev` serves it at `http://localhost:8787`. Free Cloudflare plans allow 100,000 Worker requests per day, and caching (below) keeps real usage far lower.

**CLI commands used** (prefix with `npx wrangler@3` instead of `npx wrangler` on Node 18/20)

| Command | What it does |
|---|---|
| `npx wrangler login` | Authorizes wrangler with your Cloudflare account (one time) |
| `npx wrangler deploy` | Uploads `worker/index.js` using `worker/wrangler.toml` |
| `npx wrangler dev` | Runs the Worker locally on port 8787 |
| `npx wrangler tail` | Streams live logs from the deployed Worker |
| `npx wrangler secret put ALGOLIA_KEY` | Stores the catalog search key as a secret instead of in the file |

### Config (`worker/wrangler.toml`)

| Variable | Default | Purpose |
|---|---|---|
| `ALLOWED_ORIGIN` | `*` | Value of `Access-Control-Allow-Origin`. Once everything works, set it to your site, e.g. `https://codymkw.github.io`, so other sites can't use your Worker |
| `ALGOLIA_APP` | built-in | Catalog search application ID. Override only if Nintendo changes it |
| `ALGOLIA_INDEX` | `ncom_game_en_us` | Catalog search index |
| `ALGOLIA_KEY` | built-in | Search-only key. Prefer `wrangler secret put` if you override it |

The built-in search credentials are the public, search-only ones nintendo.com's own pages use (they appear in several open-source projects). They are unofficial and Nintendo can rotate them at any time.

### Worker endpoints

All endpoints are `GET`, return JSON, and include CORS headers (errors too, so the browser can read them). Errors look like `{ "error": "message" }` with status `400` (bad input) or `502` (an upstream failed).

| Endpoint | Purpose | Limits |
|---|---|---|
| `/status` | Health check of both upstreams | none |
| `/search?q=<text>&limit=8` | Catalog search | `q` 1–120 chars, `limit` 1–24 |
| `/price?ids=<nsuid,…>&country=US` | Prices for known Nintendo IDs | 1–50 numeric IDs, one country |
| `/lookup?titles=<A\|B\|C>&countries=US,CA,MX` | Title → best catalog match → prices. **This is what the site uses** | 1–12 titles (separated by `\|`), each ≤120 chars |

`country` / `countries` accept `US`, `CA`, `MX`. One North American ID prices all three.

**`/lookup` response**

```jsonc
{
  "countries": ["US", "CA", "MX"],
  "results": [                       // same order as the titles you sent
    {
      "query": "The Legend of Zelda: Breath of the Wild",
      "match": {                     // null when no confident match was found
        "nsuid": "70010000000025",
        "title": "The Legend of Zelda™: Breath of the Wild",
        "url": "https://www.nintendo.com/us/store/products/…",
        "image": "https://…", "platform": "Nintendo Switch",
        "exact": true                // false = similar but not identical title
      },
      "prices": {                    // only countries that returned a price
        "US": { "status": "onsale", "regular": 59.99, "currency": "USD",
                "sale": null, "saleStarts": null, "saleEnds": null },
        "CA": { "status": "onsale", "regular": 79.99, "currency": "CAD", "sale": 59.99,
                "saleStarts": "2026-10-10T00:00:00Z", "saleEnds": "2026-10-25T06:59:59Z" }
      }
    }
  ]
}
```

`status` comes straight from Nintendo: `onsale` (purchasable; says nothing about discounts, check `sale`), `unreleased`, `not_found`, `sales_termination`, and others. `regular` and `sale` are numbers, and `sale` is `null` unless there's a real discount (a "discount" equal to or above the regular price is ignored).

**`/price` response**: `{ "country": "US", "prices": { "<nsuid>": { …same price object… } } }`. IDs Nintendo doesn't know come back as `{ "status": "not_found", … }`.

**`/search` response**: `{ "results": [ { nsuid, title, image, url, platform, releaseDate, esrb, msrp, salePrice } ] }`.

**`/status` response**

```jsonc
{
  "ok": true,                                   // both checks passed
  "search": { "ok": true, "hits": 3, "sample": "Mario Kart™ 8 Deluxe", "error": null },
  "price":  { "ok": true, "sample": { "status": "onsale", "regular": 59.99, … }, "error": null },
  "config": { "algoliaApp": "…", "algoliaIndex": "ncom_game_en_us" }   // the key is never returned
}
```

### How matching works

Vantage's catalog comes from RAWG, but Nintendo prices are looked up by Nintendo's own ID, so the Worker searches Nintendo's catalog for each title and accepts a result only if it's a confident match: titles are lowercased, `™ ® ©` and punctuation are removed, a leading "The" is dropped, and the word overlap must be at least 75%. A wrong price is worse than no price, so `Hades` will **not** match `Hades II`. When the match is similar but not identical, the game page says "Matched to …" so you can spot a wrong game.

### Caching

| What | Cached for | Where |
|---|---|---|
| Catalog search results | 24 hours | Cloudflare edge cache (in the Worker) |
| Prices | 1 hour | Cloudflare edge cache (in the Worker) |
| `/lookup` responses | 30 minutes | The visitor's browser (`localStorage`, same cache as the other APIs) |

So a price can be up to roughly 1.5 hours stale, and the site says to confirm on the store page before buying.

### Troubleshooting the Worker

| Symptom | Likely cause and fix |
|---|---|
| Cards say **Price lookup failed** | The Worker is down or `PRICE_API` is wrong. Open `<PRICE_API>/status` in a browser; check there's no trailing slash |
| Browser console shows a CORS error for the Worker URL | `ALLOWED_ORIGIN` doesn't match your site's origin (scheme + host, no path). Use `*` while testing |
| `/status` → `search.ok: false`, error mentions `403` or `401` | Nintendo rotated the catalog search key. Find the current app ID, key and index in the network tab on nintendo.com's game search, then set `ALGOLIA_APP`, `ALGOLIA_INDEX` and (as a secret) `ALGOLIA_KEY` |
| `/status` → `price.ok: false` | Nintendo's price endpoint is failing or changed. Check `error`; try again later |
| **No eShop price found** on a game | The title didn't match confidently, or it isn't sold on the US eShop (RAWG and Nintendo name things differently). Not a bug in the Worker |
| Price is missing for one country only | That country returned `not_found` (the game isn't sold there) |
| A game shows "Matched to …" with a different name | Check the store link; if the match is wrong, the title can't be matched reliably and the price should be ignored |

## Client-side storage

| Key | Where | Contents |
|---|---|---|
| `vantage:wish` | `localStorage` | Wishlist items (with the price when added, for drop detection) |
| `vantage:cart` | `localStorage` | Cart items |
| `vantage:filters` | `localStorage` | Last-used Browse filters (title is never saved) |
| `vantage:ninView` | `localStorage` | Nintendo page layout: `"grid"` or `"list"` |
| `vantage:rawgKey` | `localStorage` | RAWG key pasted into the Nintendo setup card (not used if `RAWG_KEY` is set) |
| `vantage:c:<url>` | `localStorage` | API response cache. Fresh for 5–10 min (stores 24h), then kept as a fallback if the network fails. Cleared automatically if storage fills up |

Clear site data to reset everything. Wishlist and cart are per-browser, with no accounts or backend.

## Routes

Hash-based, so it works on any static host:

| Hash | View |
|---|---|
| `#/` | Discover |
| `#/browse` | Browse with filters |
| `#/nintendo` | Nintendo catalog |
| `#/free` | Free game giveaways |
| `#/wishlist` | Wishlist |
| `#/game/<gameID>` | PC game detail modal (CheapShark ID) over the current page |
| `#/nin/<rawgID>` | Nintendo game detail modal (RAWG ID) over the current page |

## Keyboard shortcuts

| Key | Action |
|---|---|
| `/` | Focus search |
| `↑` / `↓` | Move through suggestions |
| `Enter` | Open suggestion, or search all results |
| `Esc` | Close modal, cart, or suggestions |

## Customizing

Everything lives in `index.html`:

- **RAWG key** and **price Worker URL**: the `RAWG_KEY` and `PRICE_API` constants at the top of the script
- **Giveaway filters**: `GP_PLATS` and `GP_TYPES`; **Nintendo tabs**: `NIN_TABS`
- **Name and branding**: the `<title>`, `.logo` in the header, and the footer text
- **Colors**: CSS variables in `:root` (`--acc`, `--acc2`, `--bg`, `--panel`, …)
- **Home rows**: edit the `ROWS` array (each row is a title plus CheapShark query params)
- **Sort options**: edit `SORTS`
- **"See all" presets**: edit `PRESETS`
- **Hero picks**: logic in `loadHome()` (currently Metacritic ≥ 80 and ≥ 25% off, top 5)

## Limitations

- Prices are **USD only** and update on CheapShark's schedule (roughly every couple of hours), not truly real-time
- The cart is a comparison cart: **checkout opens each store's page**, since Vantage doesn't sell anything itself. Browsers may ask to allow multiple tabs
- Games sold only outside Steam get a smaller thumbnail instead of full artwork
- The Nintendo section needs a RAWG key. Nintendo **prices** also need the price Worker; without it the section shows ratings and store links only
- Nintendo prices cover the **US, Canada and Mexico** (one North American ID prices all three). Europe, the UK, Japan and Australia use different IDs and aren't supported
- Nintendo has no price history: Vantage shows today's price and any active sale, not a chart. The wishlist flags a **price drop** relative to the price the first time it saw the game
- Nintendo prices rely on **unofficial** endpoints that can change without notice (see Troubleshooting). The rest of the site keeps working if they do
- Nintendo games can be wishlisted but not added to the cart (the cart is for PC deals that link to a store)
- CheapShark rate-limits bursts of requests, and its `429` response has no CORS headers, so the browser reports a rate limit as a **CORS error** (`No 'Access-Control-Allow-Origin' header`). Vantage works around this: requests to each API go out one at a time with a short gap, failures are retried with backoff (a masked `429` shows up as a thrown `fetch` error, not a status code), results are cached in `localStorage` (5–10 min fresh, 24h for the store list), and if the network still fails the last cached copy is shown instead of an error. If you ever see a wall of CORS errors in the console, it's almost always a rate limit; wait a minute and reload
- Wishlist and cart don't sync between devices

## Credits

- Pricing data: [CheapShark](https://www.cheapshark.com)
- Nintendo catalog data: [RAWG](https://rawg.io)
- Nintendo eShop prices: Nintendo's own eShop, via the Worker. Nintendo, Switch and related names are trademarks of Nintendo; Vantage isn't affiliated with or endorsed by Nintendo
- Free game giveaways: [GamerPower](https://www.gamerpower.com)
- Artwork: Steam CDN. All game art and trademarks belong to their owners
- Font: [Inter](https://rsms.me/inter/)

## License

Add a license of your choice (MIT is a good default). Game data and artwork are not covered by it.
