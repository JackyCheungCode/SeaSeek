# SeaSeek

Modular global business intelligence and lead discovery platform.

## Current foundation
- Cloudflare Worker serves the SeaSeek workspace shell and API endpoints.
- Supabase schema contains organization-scoped ingestion tasks, actor runs, ads, Facebook pages, leads, contacts, source catalog, and source settings.
- Data sources are modular catalog entries. New connectors should implement connection test, input schema discovery, parameter validation, execution, normalization, and persistence.
- First connectors: Apify Facebook Ads Scraper and Facebook Pages Scraper.
- Reserved connectors: Google Search, Instagram Ads, TikTok Ads, LinkedIn, ecommerce, business directories.

## Development branch
`feature/modular-ingestion-apify-settings`

## Cloudflare Worker
Worker: `seaseek`
- `GET /api/health`
- `GET /api/sources`
- `GET /api/apify/status`
- `POST /api/apify/test` accepts a token transiently, verifies Apify account access and checks both Actor input-schema endpoints. Token is not persisted or logged.
- `GET /api/apify/schemas?actor=ads|pages` returns a sanitized live Input Schema from the configured Worker Secret.
- `POST /api/apify/validate-input` validates task parameters against the live schema on the server and rejects unknown fields.

## Apify configuration
The SeaSeek settings page provides an API token input and connection test. Configure a Cloudflare Worker Secret named `APIFY_API_TOKEN` under Worker Settings → Variables and Secrets. Never use a plain-text Worker variable or commit API tokens to Git. The previous plain-text binding was removed during the latest deployment; add a fresh Secret before using live schema loading or server-side validation.

Actor IDs:
- `apidojo/facebook-ads-scraper`
- `themineworks/facebook-pages-scraper`

The task page now generates parameter controls from the live Actor Input Schema: enum fields use select controls, booleans use checkboxes, bounded numeric fields use constrained number inputs, and array/object fields use JSON editors. Required fields, type constraints, enum values, and numeric/string bounds are checked again by the Worker. The current task action validates parameters only; it does not start a billable Actor run or persist results.

## Database
Supabase project ref: `utmcexlopdkiaxlqqdnt`.
Organization membership and RLS policies are enabled. Sensitive API credentials must not be stored in plain-text application tables. `source_settings.settings` is for non-secret metadata only.

## Deployment status
The latest Worker upload/deployment succeeded through the Cloudflare API. The prior plain-text APIFY_API_TOKEN binding was removed; no Worker Secret is currently configured, so live Schema endpoints will return a configuration error until a new Secret is added. CI/CD, authenticated frontend integration, Actor execution, run polling, and result persistence are not yet configured. The current workspace is a schema-driven configuration and validation foundation, not yet a completed ingestion runner.


## Workspace navigation and Apify settings
- Mobile navigation uses a slide-in sidebar with an overlay.
- All Facebook collection entry points are grouped under one **Facebook 采集** menu; Ads and Pages are selected inside the collection workspace.
- Apify connection and Actor selection are centralized in **Apify 设置**.
- Actor IDs can be edited and saved in the current browser profile; the saved values are used for live schema loading and server-side input validation.
- The API Token field is used for a transient connection test only. It is deliberately not persisted in browser storage or exposed through an unauthenticated save endpoint. For durable server-side use, configure the Worker Secret `APIFY_API_TOKEN` in Cloudflare Dashboard. A secure in-app credential save requires authenticated user/organization access and is not yet implemented.


## Single-user Apify credential vault
- The Apify settings screen supports testing, saving, unlocking, updating, and deleting the API token.
- The token is encrypted in the browser with AES-GCM (PBKDF2-SHA-256, 310,000 iterations) before it is stored in the Cloudflare KV namespace bound as `APIFY_CONFIG`.
- The vault passphrase is never persisted by SeaSeek. After a page reload, enter it to unlock the saved token; the token is held only in page memory while the session remains open.
- Replacing or deleting an existing credential requires unlocking it first. Keep the passphrase safe: it cannot be recovered.
- Facebook Ads and Pages Actor IDs are saved server-side in the same KV namespace.
- This is a single-user deployment, not a multi-tenant credential manager. Add Cloudflare Access in front of the Worker if the service is exposed publicly; the encrypted credential itself is not readable from the KV status endpoint.


## Facebook collection integration (current)

The Facebook workspace now exposes schema-driven Apify collection runners for:
- Pages: `topapi/facebook-pages-search-scraper` (existing keyword/location page discovery).
- Posts, videos/Reels search, events and places: `coregent/facebook-search-scraper` with server-fixed `searchType` values (`posts`, `videos`, `events`, `places`).
- Public group posts, post comments and URL-based Reels: `pnda/facebook-scraper` with server-fixed modes (`groupPosts`, `comments`, `reels`).
- Public Meta Ad Library ads: `techbellys/facebook-ads-scraper`.

All non-Page forms are generated from each Actor's live Apify OpenAPI input schema. Actor IDs and mode selectors are fixed server-side. Cookie, password, token, session and credential fields are removed/rejected; SeaSeek does not collect private groups or bypass login walls. Each run is recorded in `APIFY_CONFIG` KV, and successful datasets are archived there with source Actor, collection type, run ID, timestamp and raw output (up to the configured archive cap). The task center and data center read these records.

**Verification boundary:** Worker version 37 is deployed with the `APIFY_CONFIG` binding. Cloudflare accepted the module upload and deployment. The Apify account verification request from the integration environment returned HTTP 403, so live Actor schema retrieval and billable end-to-end runs remain pending a successful Apify API response. Do not mark any Actor as runtime-verified until its live schema and a user-approved sample run have succeeded.

## LinkedIn-related external market intelligence (fixed Apify Actor)

- A dedicated workspace is available at `/data-collection/linkedin`.
- The workspace uses one fixed Apify Actor: `apify/google-search-scraper`. Users cannot supply or change the Actor ID.
- The integration searches third-party public websites for company news, product launches, industry reports, partner announcements, and market trends relevant to LinkedIn-oriented B2B research.
- Search queries automatically append `-site:linkedin.com -site:lnkd.in`; user-supplied `site:` operators and LinkedIn domains are rejected. Returned dataset items are filtered again before archiving.
- Actor input schema is fetched live from Apify before the run is enabled. The integration requires a text `queries` field and refuses to run if the live schema differs.
- Runs are stored under `linkedin_task:` in the existing `APIFY_CONFIG` KV namespace. Successful external search results are archived under `linkedin_data:`, with Actor ID, query, Run ID, collection timestamp, and raw items.
- The integration does **not** scrape LinkedIn pages, member profiles, posts, comments, private groups, or ad accounts; it does not use LinkedIn cookies or bypass login/access controls. LinkedIn-origin domains are excluded from queries and filtered from returned results.
- This is Google-indexed third-party public web research about LinkedIn-related topics, not LinkedIn platform data access. Live schema validation and a sample run must succeed before describing it as runtime-verified.
- Fixed Actor reference: [Apify Google Search Scraper](https://apify.com/apify/google-search-scraper).
