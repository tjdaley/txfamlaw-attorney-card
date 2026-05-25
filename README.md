# TXFamLaw Attorney Cards & Directory

A WordPress plugin that renders attorney contact cards, a searchable attorney
directory, structured profile lists (credentials, education, awards, etc.), and
`ProfilePage`/`Person` schema — all driven by the txfamlaw attorney API. Data is
fetched server-side and cached, so pages stay fast and the API isn't hit on every
view.

**Current version:** 2.6.0

---

## What it does

- **Contact card** — email, phone, office, social links, and an optional
  "Schedule a Consultation" button.
- **Directory** — a searchable grid of attorneys with thumbnails, linking to each
  attorney's profile page.
- **Profile lists** — styled, headed bullet lists for credentials, education,
  professional involvement, awards, and publications.
- **Schema** — emits a clean `ProfilePage` → `Person` JSON-LD graph (with
  `jobTitle`, `address`, `sameAs`, `worksFor`, `knowsAbout`, `hasCredential`,
  `alumniOf`, `memberOf`, `award`) from the same API data, and suppresses Rank
  Math's competing schema on attorney pages.

All output pulls brand colors from the Kadence global palette, with hex fallbacks.

---

## Shortcodes

### Contact card
```
[attorney_contact slug="tjd"]
```
- `slug` (required) — the attorney's API slug.
- `format` — `card` (default) or `directory` (renders a single directory tile).

### Directory
```
[attorney_directory]
```
- `search` — `yes` (default) or `no` to hide the search box.
- `office` — optional pre-filter, e.g. `office="Plano"`.

Lists every attorney whose `include_in_directory` is true, sorted by `order_by`.

### Profile lists
```
[attorney_list slug="tjd" type="credentials"]
[attorney_list slug="tjd" type="awards" limit="8"]
[attorney_list slug="tjd" type="education" heading="Education"]
```
- `slug` (required) — the attorney's API slug.
- `type` (required) — one of: `credentials`, `degrees`, `organizations`,
  `awards`, `publications`.
- `heading` — optional custom section title (sensible default per type).
- `limit` — optional cap on the number of **visible** items (a "+ N more" note
  is shown). Schema always includes the full list regardless of `limit`.

An empty or absent list renders nothing — no empty heading.

---

## Caching & refresh

- Each attorney and the directory list are cached for 6 hours (WordPress
  transients).
- Admins see a **"↻ Refresh Attorney Data"** button in the WordPress admin bar.
  Clicking it clears all attorney caches so the next page view fetches fresh
  data. It's nonce-protected and visible only to users who can manage options.
- Deactivating the plugin also clears all caches.

---

## Installation

This plugin is intended to be baked into the site's Docker image (placed in
`wp-content/plugins/txfamlaw-attorney-card/`) and activated once. It can also be
installed by uploading the zip via **Plugins → Add New → Upload Plugin** for
quick testing. See `DEPLOYMENT.md` in the plugin folder for the full deployment
notes specific to the multi-server + shared-RDS setup.

After install, activate the plugin once. Activation state lives in the shared
database, so a single activation applies across all servers.

---

## Data source

The plugin reads from `https://txfamlaw.com/api/attorneys` (list) and
`https://txfamlaw.com/api/attorneys/{slug}` (single). The API base URL is defined
near the top of `txfamlaw-attorney-card.php` if it ever needs to change.

The visible page and the schema are only as good as the API data. Populate
attorney records fully (bio, socials, credentials, etc.) and preview a page
before publishing.

---

## Files

- `txfamlaw-attorney-card.php` — main plugin: data layer, shortcodes, admin
  refresh button.
- `attorney-schema.php` — ProfilePage/Person schema emitter and Rank Math
  suppression.
- `attorney-card.css` — styling for cards, directory, and lists.
- `attorney-directory.js` — client-side directory search.
- `DEPLOYMENT.md` — deployment notes for the Docker/multi-server workflow.

---

## Notes

- Social values that aren't valid URLs are skipped (both in the card and in
  `sameAs` schema).
- On attorney pages, the plugin suppresses Rank Math's own schema to avoid a
  duplicate entity; Rank Math continues to handle everything else (meta, OG
  tags, breadcrumbs, sitemaps) on those pages and all schema on non-attorney
  pages.
- Requires the Kadence theme for palette colors (falls back to brand hex values
  if absent) and Font Awesome for icons (loaded from CDN if the theme doesn't
  already provide it).
