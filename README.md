# Vantage

A sleek, single-file game storefront that compares live PC game prices across Steam, GOG, Humble, Fanatical, Epic and more. No build step, no framework, no API key: just one `index.html`.

Built by **CodyMKW** and **Joseph** as a vibe-coded take on the Epic Games Store layout, with better game listings.

> Vantage is an independent fan project. It is not affiliated with Epic Games, Valve, or any storefront. Purchases always complete on the store's own site.

---

## Features

- **Discover page**: auto-rotating hero banner with a side list of featured picks, plus scrollable rows: *Free right now*, *Top deals*, *Critically acclaimed*, *Great games under $10*, *Deepest discounts*
- **Portrait game cards** with discount badges, strike-through original price, and a wishlist heart
- **Browse page** with filters: sort, store, max price, minimum Metacritic, on-sale only, AAA only, and Load more pagination
- **Live search** with suggestions as you type (arrow keys + Enter supported)
- **Game detail view**: hero art, Metacritic and Steam ratings, release date, publisher, price comparison across every store, lowest price ever, and links to Steam and Metacritic
- **Wishlist** that refreshes prices on open and flags **price drops**
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

Vantage uses two free, keyless sources:

- **[CheapShark API](https://apidocs.cheapshark.com/)** for prices, deals, store info, and ratings
- **Steam CDN** for artwork: `https://cdn.cloudflare.steamstatic.com/steam/apps/<appid>/…`

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
| `vantage:c:<path>` | `sessionStorage` | API response cache (5–10 min; stores 24h) |

Clear site data to reset everything. Wishlist and cart are per-browser, with no accounts or backend.

## Routes

Hash-based, so it works on any static host:

| Hash | View |
|---|---|
| `#/` | Discover |
| `#/browse` | Browse with filters |
| `#/wishlist` | Wishlist |
| `#/game/<gameID>` | Game detail modal over the current page |

## Keyboard shortcuts

| Key | Action |
|---|---|
| `/` | Focus search |
| `↑` / `↓` | Move through suggestions |
| `Enter` | Open suggestion, or search all results |
| `Esc` | Close modal, cart, or suggestions |

## Customizing

Everything lives in `index.html`:

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
- CheapShark rate-limits heavy use; the app retries on `429` and caches responses, but don't hammer it
- Wishlist and cart don't sync between devices

## Credits

- Pricing data: [CheapShark](https://www.cheapshark.com)
- Artwork: Steam CDN. All game art and trademarks belong to their owners
- Font: [Inter](https://rsms.me/inter/)

## License

Add a license of your choice (MIT is a good default). Game data and artwork are not covered by it.
