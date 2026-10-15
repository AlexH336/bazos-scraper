# Bazos.sk — site inspection notes

Hand inspection before writing any scraper code (Step 1). Last checked: 2026-10-15.

## TL;DR

- Category list pages are plain static HTML → `requests` + BeautifulSoup is enough.
- Category and pagination paths are allowed by robots.txt. **Search and filter URLs are not** (`hledat=`, `cenaod=`, `cenado=`, `order=`, …).
- Scrape **list pages only** (≈20 listings per request), not detail pages.
- Listing ID in the URL is the unique key.
- Requesting a page past the end returns page 1, not a 404 → pagination needs an explicit stop condition.
- Never store seller data (name, phone, user ID, coordinates).

## 1. robots.txt

Each subdomain has its own robots.txt. The one that applies to us is the category subdomain, e.g. `https://auto.bazos.sk/robots.txt`.

Relevant rules for `User-agent: *` (auto subdomain):

```
Disallow: /*hledat=        # search query
Disallow: /*hlokalita=     # location filter
Disallow: /*humkreis       # radius filter
Disallow: /*order=         # sorting
Disallow: /*cenaod=        # price from
Disallow: /*cenado=        # price to
Disallow: /*rubriky=
Disallow: /*type=
Disallow: /*category=
Disallow: /ad-phone.php    # seller phone
Disallow: /ad-mail.php     # seller email
```

The rest are internal actions (`suggest.php`, `insert.php`, `upload.php`, `favourite.php`, `zmazat/`, …). The `www` robots.txt has a similar but longer list (also `search.php`, rating/report pages, payments).

Named bots blocked completely: `trovitBot`, `pricebot`, `magpie-crawler`, `SemrushBot`, `trendictionbot`, `BLEXBot`.

The auto subdomain also lists `Sitemap: https://auto.bazos.sk/sitemap.php`.

**Conclusions**

- Category paths (`/bmw/`) and pagination (`/bmw/20/`) are **allowed**.
- Search, price filters and sorting are **disallowed**. For specific products (e.g. "BMW 540i G30"), fetch the category path and filter titles in our own code.
- `pricebot` is blocked outright, so the site doesn't want price-comparison crawlers. That doesn't technically apply to us, but it's a reason to stay small: few pages, long delays, cached responses, clear User-Agent.

## 2. Terms of use

- URL: https://www.bazos.sk/podmienky.php
- Nothing about automated access, robots or scraping.

## 3. API / RSS / sitemap

- **API:** official API exists only on request, with permission. Not pursued.
- **RSS:** available per category, e.g. `https://www.bazos.sk/rss.php?rub=hu&cat=287` (linked from the page `<head>`). **Doesn't include prices**, so it can't replace HTML scraping.
- **Sitemap:** `https://auto.bazos.sk/sitemap.php`. Not inspected yet.

## 4. URLs and pagination

- Category = subdomain: `<category>.bazos.sk`, e.g. `auto.bazos.sk`
- Subcategory = path: `auto.bazos.sk/fiat/`
- Page 1 has no number. Each later page adds an offset in steps of 20:

| Page | URL |
|---|---|
| 1 | `auto.bazos.sk/fiat/` |
| 2 | `auto.bazos.sk/fiat/20/` |
| 3 | `auto.bazos.sk/fiat/40/` |

- 20 listings per page. The last page works the same way, even with 1 listing.
- **Edge case:** an offset past the end (e.g. `/fiat/99980/`) returns **page 1**, not a 404.

**Stop-condition options (decide in Step 4)**

- [x] Total listing count is shown in a header row above the listings: `Zobrazených 1-20 inzerátov z 8 690` → last offset can be computed. Same number format as prices (spaces / possible NBSP as thousands separator).
- [ ] Stop when a page has fewer than 20 listings.
- [ ] Stop when a page only contains listing IDs we've already seen.
- [ ] Hard cap on pages as a safety net, whatever else we choose. 8 690 BMW listings ≈ 435 pages, so the demo only scrapes the first few.

**Demo category:** `https://auto.bazos.sk/bmw/`

**Header row trap:** the header row reuses the card's inner classes:

```html
<div class="listainzerat inzeratyflex">
  <div class="inzeratynadpis">... Zobrazených 1-20 inzerátov z 8 690</div>
  <div class="inzeratycena"><b><span class="paction">Cena</span></b></div>
  <div class="inzeratylok">Lokalita</div>
  <div class="inzeratyview"><span class="paction">Zobrazenie</span></div>
</div>
```

Selecting `.inzeratycena` or `.inzeratyflex` across the whole page also matches this header ("Cena"). Select cards with `.inzeraty` first, then look inside each card.

## 5. List page HTML

One card per listing, all inside `div.maincontent`:

```
div.inzeraty.inzeratyflex          ← one listing (anchor selector: .inzeraty)
├── div.inzeratynadpis
│   ├── a > img.obrazek            ← thumbnail (same href as title)
│   ├── h2.nadpis > a              ← title + relative URL   → use this
│   ├── span.velikost10            ← date, plus span.ztop if it's a TOP listing
│   └── div.popis                  ← shortened description
├── div.inzeratycena               ← price (raw text)
├── div.inzeratylok                ← city <br> postal code
├── div.inzeratyview               ← view count, "720 x"
└── div.inzeratyakce               ← action buttons (ignore; contain seller user ID)
```

Trimmed sample (seller data removed):

```html
<div class="inzeraty inzeratyflex">
  <div class="inzeratynadpis">
    <a href="/inzerat/196305315/bmw-x6-e71-35d-210-kw.php"><img class="obrazek" ...></a>
    <h2 class="nadpis"><a href="/inzerat/196305315/bmw-x6-e71-35d-210-kw.php">Bmw X6 E71 35d 210 kw</a></h2>
    <span class="velikost10"> - <span title="TOP 1x Platí do 15.10. 2026" class="ztop">TOP</span> - [15.10. 2026]</span><br>
    <div class="popis">Predám Bmw X6 E71 35d 210 kw, r.v. 2009, 281000 km, ...</div>
  </div>
  <div class="inzeratycena"><b><span translate="no">  12 000 €</span></b></div>
  <div class="inzeratylok">Martin<br>036 01</div>
  <div class="inzeratyview">720 x</div>
  <div class="inzeratyakce">...</div>
</div>
```

**Notes for Step 2**

- Use `.inzeraty` as the card anchor, not a long `body > div > …` chain.
- Use `h2.nadpis a` for title + link (the thumbnail has a second `<a>` with the same href).
- Links are **relative** → join them with the subdomain base URL.
- **Listing ID** = the number in the URL (`/inzerat/196305315/…`) → unique key for deduplication.

**TOP listings**

Regular:

```html
<span class="velikost10"> - [15.10. 2026]</span>
```

TOP (paid promotion):

```html
<span class="velikost10"> - <span title="TOP 1x Platí do 15.10. 2026" class="ztop">TOP</span> - [15.10. 2026]</span>
```

- [x] TOP listings didn't repeat across the pages checked. Still dedupe by listing ID as a safety net.

## 6. Price formats

Raw text examples (note the leading spaces; the thousands separator may be a non-breaking space):

| Raw | Meaning | Parsed value |
|---|---|---|
| `  23 900 €` | price | `23900` |
| `Zadarmo` | free | `0` |
| `Dohodou` | negotiable | `None` |
| `Ponúknite` | make an offer | `None` |
| `V texte` | price in description | `None` |

Plan: keep the raw string **and** the parsed number. Parse in a separate function.

## 7. Personal data

Seller data on **detail pages**: name, phone (hidden behind a click + mobile verification), approximate GPS location, postal code, city.
Seller data on **list pages**: city + postal code; seller user ID inside `inzeratyakce`.

**Decision**

- **Store:** listing ID, title, price (raw + parsed), URL, city, date, TOP flag, views.
- **Never store:** seller name, phone, email, user ID, GPS coordinates.

## 8. Static vs JS-rendered

**Static.** Listing titles and descriptions are present in View Source (not only in DevTools), so `requests` receives the full content and no browser automation is needed.
