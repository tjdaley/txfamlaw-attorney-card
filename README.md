# TXFamLaw Attorney Cards & Directory

A WordPress plugin that renders attorney contact cards, a searchable attorney
directory, structured profile lists (credentials, education, awards, etc.),
`ProfilePage`/`Person` schema, and a lead-capture contact form — all driven by
the txfamlaw attorney API. Data is fetched server-side and cached, so pages stay
fast and the API isn't hit on every view.

**Current version:** 2.9.1

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
- **Contact form** — a "Schedule a Consultation" lead form that submits
  server-side (proxied through WordPress) to the FastAPI leads endpoint, with
  attorney attribution, spam defense, and visitor context for geolocation.

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

**Ordering.** Within each list, items are ordered **pinned first, then most
recent, then undated last**. An item with `"pinned": true` always appears (it
survives the `limit`) and sorts to the top. Among unpinned items, the most
recent year wins; items with no parseable year sort last. This means an attorney
can force a favorite award (or an important but undated item) to always show by
pinning it. The date field used for sorting is per type: `year` for awards,
`date` for publications, `since` for organizations and credentials. Pinning
affects the **visible list only**; the schema always emits every item.

#### Sequence of Lists

My opinion is that lists should be shown in this order:

- Degrees (no limit)
- Credentials (no limit)
- Publications (limit 10)
- Awards (limit 10)
- Organizations (limit 10)

---

## Contact form

```
[attorney_contact_form]
[attorney_contact_form slug="nag"]
[attorney_contact_form show_picker="no" heading="Talk to Us" subhead="..."]
```

Renders a "Schedule a Consultation" lead form. The browser submits same-origin
to WordPress, which validates and **proxies** the lead server-side to the
FastAPI endpoint — the endpoint and API key are never exposed to the browser.

- `slug` — optional. Pins the lead to a specific attorney (use on bio pages).
- `show_picker` — `yes` forces the "Who would you like to reach?" dropdown;
  `no` suppresses it; omitted = auto (shown only when no `slug` is set).
- `heading` / `subhead` — optional copy overrides. If omitted and a `slug` is
  set, the subhead names that attorney.

### Attorney attribution (who gets the lead)

Resolved server-side, most specific wins:

1. The visitor's explicit dropdown choice
2. The form's `slug` attribute (bio-page embed)
3. The subdomain (e.g. `nag.txfamlaw.com` → `nag`; `www`/`staging` are ignored)
4. The post author's `user_nicename` (for content pages — mapped to an attorney
   on the FastAPI side)
5. `www` (the synthetic/default attorney)

The resolved value is sent as `attorney_slug`.

### What the form sends

Matches the FastAPI `Lead` model: `attorney_slug`, `url_path`, `full_name`,
`email`, `telephone`, `audit_score`, `needs_follow_up`, `ip_address`,
`user_agent`, `referrer`, `request_headers` (a curated subset — accept-language,
user-agent, referer, client hints; **no cookies**), `conflict_summary`, and
`lead_source` (`"wordpress"`). `session_uuid` is omitted.

**Geolocation:** `country` / `state` / `city` / `zip` are **not** sent — the
FastAPI server fills these from the `ip_address` field in the payload. Because
the lead is proxied, FastAPI must geolocate from `lead.ip_address` (the
forwarded visitor IP), **not** from the request's connection IP (which is now
the WordPress/ALB server). The plugin reads the real visitor IP from
`X-Forwarded-For` (left-most entry), falling back to `REMOTE_ADDR`.

### Spam defense

- A hidden honeypot field (`website`) — if filled, the submission is silently
  dropped and nothing is forwarded.
- A WordPress nonce (CSRF protection).
- A per-IP rate limit (5 submissions per 10 minutes), keyed off the real
  visitor IP.

### Required configuration (`wp-config.php`)

```php
// API key sent to the leads endpoint (kept server-side, never in the browser).
define( 'TXFL_LEADS_API_KEY', 'your-secret-key-here' );

// Optional: override the leads endpoint (defaults to the constant below).
define( 'TXFL_LEADS_ENDPOINT', 'https://txfamlaw.com/api/leads' );
```

If `TXFL_LEADS_API_KEY` is defined, it is sent in the payload as `api_key`.

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
```

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
**title** | The title of the publication or presentation | AI Evidence in Texas Litigation | **YES** | NONE
**venue** | Where it was published or presented | Texas Bar Journal | **YES** | NONE
**date** | When it was published or presented | 2025-07-01 | NO | NONE
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
- `attorney-contact-form.php` — "Schedule a Consultation" form and the
  server-side proxy to the FastAPI leads endpoint.
- `attorney-card.css` — styling for cards, directory, lists, and the form.
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
