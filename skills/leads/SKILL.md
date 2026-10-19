---
name: leads
description: Search local businesses by niche and city via the LeadMerlin v2 API, enrich them (owner names, verified emails, phone line types, website intelligence), and export CRM-ready CSVs. Use when the user asks for lead generation, prospect lists, local business research, or domain enrichment.
---

# /leads — LeadMerlin lead generation

Search and enrich local business leads using the LeadMerlin v2 API,
then export the results to CSV.

## Configuration (never hardcode the key)

Read both values from the environment — ask the user once if missing and tell
them to export it in their shell profile or put it in the project `.env`:

- `LEADMERLIN_API_KEY` — API key, starts with `lm_` (get it at https://app.leadmerlin.com/dashboard). Send as the `x-api-key` header on
  every request. NEVER write the key into files, code, or this skill.
- `LEADMERLIN_API_BASE` — base URL. Default: `https://api.leadmerlin.com/api`.
  Local dev: `http://localhost:8090/api`.

```bash
BASE="${LEADMERLIN_API_BASE:-https://api.leadmerlin.com/api}"
AUTH=(-H "x-api-key: $LEADMERLIN_API_KEY")
```

## Endpoints (v2)

- Locations: `GET $BASE/v2/locations?q=<city>` — fuzzy search, returns `location_code` values
- Suggest: `GET $BASE/v2/suggest?niche=<phrase>` — free; returns `categories[]`
  (Google Business Profile categories) and `keywords[]` (sharper Maps search
  phrases). Use it whenever the user's niche is broad or one generic word.
- Search: `POST $BASE/v2/searches` — body `{ "keyword": "...", "location_codes": [<code>, ...], "strictness"?: "off"|"moderate"|"strict" }` (1–5 codes, multi-city in one job) → returns `job_id`. `moderate` (default) drops keyword-coincidence results (bars, rentals) without charging credits; `strict` also drops adjacent trades; `off` keeps everything Google returns.
- Poll: `GET $BASE/v2/jobs/:id` — poll every 5s until `status === "completed"`
- Results: `GET $BASE/v2/jobs/:id/results?cursor=<id>&limit=200` — **paginated**: follow `next_cursor` until `has_more` is false
- Enrich: `POST $BASE/v2/enrichments` — body `{ "leads": [{ "domain", "business_name"?, "city"?, "state"? }] }` (up to 100 per request, async) → returns `job_id`, same poll + results flow
- Usage: `GET $BASE/v2/usage` — plan + remaining lead credits
- Demo: `$BASE/v2/demo/*` mirrors (locations, searches, jobs/:id, jobs/:id/results, enrichments) — no auth, no credits, responses carry `demo: true`. Use these to test the flow before spending credits.

## Credits

One pool of lead credits. Each enrichable business found via search, or each
lead sent to `/v2/enrichments`, costs 1 credit. Listings without a website are
free (`skipped`), and off-niche listings filtered out by the niche classifier
are free too (`off_niche_count` on the job). Failed enrichments are refunded
automatically. Searches require a balance of 100+ to start. Polling, locations,
and suggest are free.

## Workflow

1. If the niche is one generic word (e.g. "basement", "spa"), call
   `GET /v2/suggest?niche=<phrase>` first and confirm the sharpest keyword with
   the user — generic words drag in keyword-coincidence businesses.
2. `GET /v2/locations?q=<city>` — pick the best match (prefer US City results)
3. `POST /v2/searches` with the keyword and chosen `location_codes`
4. Poll `GET /v2/jobs/:id` every 5s until `status === "completed"` (enrichment takes 30–90s+).
   The job carries `category_facets` (count per Google category) and
   `off_niche_count` — mention both in your summary.
5. Page through `GET /v2/jobs/:id/results` collecting all businesses
6. Ask the user before exporting, then write the CSV with the exact header below
7. Save as `<keyword>-<city>-leads.csv`, ready for any CRM (GHL, Instantly, HubSpot)

## CSV export (exact header order)

```csv
Google Maps Rank,Name,Category,Website,GBP URL,Google CID,Owner,Email,Phone,Phone Type,Address,City,State,Rating,Reviews,Review Snippet,Claimed,Socials,Logo URL,Main Image URL,Email 1,Email 1 Status,Email 2,Email 2 Status,Email 3,Email 3 Status,Email 4,Email 4 Status,Email 5,Email 5 Status,Phone 1,Phone 1 Type,Phone 1 Carrier,Phone 2,Phone 2 Type,Phone 2 Carrier,Phone 3,Phone 3 Type,Phone 3 Carrier,Phone 4,Phone 4 Type,Phone 4 Carrier,Phone 5,Phone 5 Type,Phone 5 Carrier,Page 1 URL,Page 1 Title,Page 1 Meta,Page 1 Text,Page 2 URL,Page 2 Title,Page 2 Meta,Page 2 Text,Page 3 URL,Page 3 Title,Page 3 Meta,Page 3 Text,Page 4 URL,Page 4 Title,Page 4 Meta,Page 4 Text,Page 5 URL,Page 5 Title,Page 5 Meta,Page 5 Text
```

Mapping from each business in `results.businesses[]`:

- `Google Maps Rank` ← `rank`; `Name` ← `name`; `Category` ← `category`
- `Website` ← `website`; `GBP URL` ← `gbp_url`; `Google CID` ← `cid`
- `Owner` ← `owner`; `Email` ← `email`; `Phone` ← `phone`; `Phone Type` ← `phone_type` (best-match promoted fields)
- `Address` ← `contact.address`; `City` ← `contact.city`; `State` ← `contact.state_code` or `contact.state`
- `Rating` ← `reputation.rating`; `Reviews` ← `reputation.reviews`; `Review Snippet` ← `reputation.review_snippet`; `Claimed` ← `reputation.is_claimed`
- `Socials` ← JSON-stringified `web.socials`; `Logo URL` ← `logo`; `Main Image URL` ← `main_image`
- `Email 1-5` / `Email 1-5 Status` ← first five of `contact.all_emails[]` (`email`, `status`)
- `Phone 1-5` / `Type` / `Carrier` ← first five of `contact.all_phones[]` (`number`, `line_type`, `carrier`)
- `Page 1-5 URL/Title/Meta/Text` ← first five of `web.pages[]` (`url`, `title`, `meta_description`, `text`)

Empty values stay empty (not "null"). Quote fields containing commas/newlines.

## Filtering and personalization (when asked)

The enriched data supports downstream selection — use it instead of guessing:
verified non-role emails (`contact.all_emails[].status === "verified" &&
!is_role_based`), mobile numbers for SMS (`line_type === "mobile"`), tech-stack
targeting (`web.tech.cms === "wordpress"`, empty `web.tech.analytics`), review
thresholds (`reputation.reviews`), owner-led outreach (`owner` not null), and
first-line personalization from `web.pages[].text` and `services`.

## Notes

- Never invent data for missing fields; leave CSV cells empty.
- On 401 ask the user to check `LEADMERLIN_API_KEY`; on 429 report the credit balance
  from the error body and stop; do not retry a failed debit in a loop.
- Status meanings: emails — verified / catch-all / invalid / unverified;
  phones — mobile / landline / voip / toll_free / unknown.
