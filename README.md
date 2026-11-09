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

## Apify configuration
The SeaSeek settings page provides an API token input and connection test. After a successful test, configure a Cloudflare Worker Secret named `APIFY_API_TOKEN` under Worker Settings → Variables and Secrets. Never commit API tokens to Git.

Actor IDs:
- `apify/facebook-ads-scraper`
- `apify/facebook-pages-scraper`

Task forms must be generated from the live Actor Input Schema. Do not hardcode unsupported Actor parameters. Enum fields should render as dropdowns; booleans as switches; bounded numeric fields as constrained inputs; required fields and allowed values must be validated server-side before execution.

## Database
Supabase project ref: `utmcexlopdkiaxlqqdnt`.
Organization membership and RLS policies are enabled. Sensitive API credentials must not be stored in plain-text application tables. `source_settings.settings` is for non-secret metadata only.

## Deployment status
The Worker is deployed directly through the Cloudflare API. CI/CD and authenticated frontend integration are not yet configured. The current workspace is an initial shell and Apify connection diagnostic, not yet a completed ingestion runner.
