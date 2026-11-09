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
- `apify/facebook-ads-scraper`
- `apify/facebook-pages-scraper`

The task page now generates parameter controls from the live Actor Input Schema: enum fields use select controls, booleans use checkboxes, bounded numeric fields use constrained number inputs, and array/object fields use JSON editors. Required fields, type constraints, enum values, and numeric/string bounds are checked again by the Worker. The current task action validates parameters only; it does not start a billable Actor run or persist results.

## Database
Supabase project ref: `utmcexlopdkiaxlqqdnt`.
Organization membership and RLS policies are enabled. Sensitive API credentials must not be stored in plain-text application tables. `source_settings.settings` is for non-secret metadata only.

## Deployment status
The latest Worker upload/deployment succeeded through the Cloudflare API. The prior plain-text APIFY_API_TOKEN binding was removed; no Worker Secret is currently configured, so live Schema endpoints will return a configuration error until a new Secret is added. CI/CD, authenticated frontend integration, Actor execution, run polling, and result persistence are not yet configured. The current workspace is a schema-driven configuration and validation foundation, not yet a completed ingestion runner.
