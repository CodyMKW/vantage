# Vantage

A sleek, single-file game storefront that compares live PC game prices across Steam, GOG, Humble, Fanatical, Epic and more. No build step, no framework, no API key: just one `index.html`.

Built by **CodyMKW** and **Joseph** as a vibe-coded take on the Epic Games Store layout, with better game listings.

> Vantage is an independent fan project. It is not affiliated with Epic Games, Valve, or any storefront. Purchases always complete on the store's own site.

---

## Features

- **Discover page**: auto-rotating hero banner with a side list of featured picks, plus scrollable rows: *Free to claim right now*, *Free on PC stores*, *Top deals*, *Top rated on Nintendo Switch* (when a RAWG key is set), *Critically acclaimed*, *Great games under $10*, *Deepest discounts*
- **Portrait game cards** with discount badges, strike-through original price, and a wishlist heart
- **Browse page** with filters: sort, store, max price, minimum Metacritic, on-sale only, AAA only, and Load more pagination
- **Live search** with suggestions as you type (arrow keys + Enter supported)
- **Game detail view**: hero art, Metacritic and Steam ratings, release date, publisher, price comparison across every store, lowest price ever, and links to Steam and Metacritic
- **Nintendo section**: the Switch (and Switch 2) catalog with Popular / Top rated / New releases / Coming soon tabs, search, ratings, screenshots, and store links (including the Nintendo eShop)
- **Free games page**: live giveaways filtered by platform (PC, Steam, Epic, GOG, Switch, PlayStation, Xbox, Android, iOS) and type (games, DLC & loot, early access)
- **Wishlist** that refreshes prices on open and flags **price drops** (Nintendo games can be wishlisted too)
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
├── index.html   # the entire app: HTML + CSS + JS
└── README.md
```

Inside `index.html`, in order:

| Section | What's in it |
|---|---|
| `<style>` | Design tokens (CSS variables), layout, cards, modal, drawer, responsive rules |
| `<body>` | Header, three views (`#home`, `#browse`, `#wishlist`), footer, modal, cart drawer, toasts |
| `<script>` | API helpers, home rows + hero, browse filters, wishlist, game detail, cart, search, router |

## Data sources

Vantage combines four sources:

| Source | Used for | Key? |
|---|---|---|
| **[CheapShark](https://apidocs.cheapshark.com/)** | PC prices, deals, store info, ratings | No |
| **Steam CDN** | PC artwork: `https://cdn.cloudflare.steamstatic.com/steam/apps/<appid>/…` | No |
| **[RAWG](https://rawg.io/apidocs)** | Nintendo Switch catalog: art, ratings, release dates, screenshots, store links | **Yes, free** |
| **[GamerPower](https://www.gamerpower.com/api-read)** | Live free-game giveaways | No |

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

There's no official free Nintendo eShop price API, so the Nintendo section shows ratings and store links but **not prices**; the game page sends you to the eShop for the current regional price. Adding live eShop prices would need a small proxy (for example a Cloudflare Worker) in front of an unofficial source.

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
| `/redirect?dealID=<dealID>` (site root) | Outbound "Buy" links |

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

## Client-side storage

| Key | Where | Contents |
|---|---|---|
| `vantage:wish` | `localStorage` | Wishlist items (with the price when added, for drop detection) |
| `vantage:cart` | `localStorage` | Cart items |
| `vantage:filters` | `localStorage` | Last-used Browse filters (title is never saved) |
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

- **RAWG key**: the `RAWG_KEY` constant at the top of the script
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
- Nintendo games have **no prices** in Vantage (see above), and the Nintendo section needs a RAWG key
- Nintendo games can be wishlisted but not added to the cart, since there's no price to total
- CheapShark rate-limits bursts of requests, and its `429` response has no CORS headers, so the browser reports a rate limit as a **CORS error** (`No 'Access-Control-Allow-Origin' header`). Vantage works around this: requests to each API go out one at a time with a short gap, failures are retried with backoff (a masked `429` shows up as a thrown `fetch` error, not a status code), results are cached in `localStorage` (5–10 min fresh, 24h for the store list), and if the network still fails the last cached copy is shown instead of an error. If you ever see a wall of CORS errors in the console, it's almost always a rate limit; wait a minute and reload
- Wishlist and cart don't sync between devices

## Credits

- Pricing data: [CheapShark](https://www.cheapshark.com)
- Nintendo catalog data: [RAWG](https://rawg.io)
- Free game giveaways: [GamerPower](https://www.gamerpower.com)
- Artwork: Steam CDN. All game art and trademarks belong to their owners
- Font: [Inter](https://rsms.me/inter/)

## License

Add a license of your choice (MIT is a good default). Game data and artwork are not covered by it.
