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
[attorney_list slug="tjd" type="degrees" heading="Education"]
```
- `slug` (required) — the attorney's API slug.
- `type` (required) — one of: `credentials`, `degrees`, `organizations`,
  `awards`, `publications`.
- `heading` — optional custom section title (sensible default per type).
- `limit` — optional cap on the number of **visible** items (a "+ N more" note
  is shown). Schema always includes the full list regardless of `limit`.

An empty or absent list renders nothing — no empty heading.

#### Sequence of Lists

My opinion is that lists should be shown in this order:

- Degrees (no limnit)
- Credentials (no limit)
- Publications (limit 10)
- Awards (limit 10)
- Organizations (limit 10)

---

## Data Shapes

### Degrees

*Degrees* have these properties:

Field | Purpose | Sample | Required? | Default
--- | --- | --- | --- | ---
**degree** | The name of the degree conferred | J.D. | **YES** | NONE
**school** | The name of the conferring institution | Texas A&M School of Law | **YES** | NONE
**year** | The year the degree was conferred | 2009 | NO | NONE
**pinned** | Whether the degree is forced to appear and at the top of the list | True | NO | False

#### Sample JSON

```json
{
  "degree": "J.D.",
  "school": "Texas A&M University School of Law",
  "year": 2009,
  "pinned": false
}

---

### Credentials

*Credentials* have these properties:

Field | Purpose | Sample | Required? | Default
--- | --- | --- | --- | ---
**name** | The name of the credential earned | Board Certified in Family Law | **YES** | NONE
**issuer** | The name of the issuing institution | Texas Board of Legal Specialization | **YES** | NONE
**category** | Subsection of the credentials display and used for SEO | Board Certification | NO | NONE
**since** | The year the credential was earned | 2014 | NO | NONE
**pinned** | Whether the credential is forced to appear and at the top of the list | True | NO | False

#### Sample JSON

```json
{
  "name": "Board Certified in Family Law",
  "issuer": "Texas Board of Legal Specialization",
  "since": "2014",
  "category": "Board Certification"
}
```

---

### Publications

*Publications* have these properties:

Field | Purpose | Sample | Required? | Default
--- | --- | --- | --- | ---
**title** | The name of the credential earned | Board Certified in Family Law | **YES** | NONE
**venue** | The name of the issuing institution | Texas Board of Legal Specialization | **YES** | NONE
**date** | The year the credential was earned | 2014 | NO | NONE
**url** | Link to the paper, presentation, or video | https://... | No | NONE
**pinned** | Whether the publication is forced to appear and at the top of the list | True | NO | False

#### Sample JSON

```json
{
  "title": "AI Evidence in Texas Litigation",
  "venue": "Texas Bar Journal",
  "date": "2025-07-01",
  "url": "https://texasbarjournal.com/articles/2025/07/id=12345",
  "pinned": True
}
```

---

### Awards

*Awards* have these properties:

Field | Purpose | Sample | Required? | Default
--- | --- | --- | --- | ---
**name** | The name of the award | Legal Advocate of the Year | **YES** | NONE
**issuer** | The name of the organization giving the award | Legal Aid of NorthWest Texas | **YES** | NONE
**year** | The year the award was given | 2025 | NO | NONE
**pinned** | Whether the award is forced to appear and at the top of the list | True | NO | False

#### Sample JSON

```json
{
  "name": "Legal Advocate of the Year",
  "issuer": "Legal Aid of NorthWest Texas",
  "year": 2025,
  "pinned": True
}
```

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
