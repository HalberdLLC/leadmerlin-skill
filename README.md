# LeadMerlin agent skill + n8n template

Search local businesses on Google Maps by niche and city, enrich them (owner
names, verified emails, phone line types, tech stack, website text) and export
CRM-ready CSVs — from Claude Code, Cursor, Codex or n8n.

Powered by the [LeadMerlin](https://leadmerlin.com/?utm_source=github&utm_medium=skill-repo) v2 API. New accounts get 100 free leads.

## 1. Get an API key

Sign up at [app.leadmerlin.com](https://app.leadmerlin.com/signup?utm_source=github&utm_medium=skill-repo) and copy your key (starts with `lm_`) from the dashboard. Then:

```bash
export LEADMERLIN_API_KEY=lm_...
```

## 2. Install the skill

**Claude Code** (project or user level):

```bash
mkdir -p ~/.claude/skills/leads
curl -sL https://raw.githubusercontent.com/HalberdLLC/leadmerlin-skill/main/skills/leads/SKILL.md -o ~/.claude/skills/leads/SKILL.md
```

Then ask: *"Find dentists in Irvine, CA with verified emails and export a CSV."*

**Cursor / Codex / other agents:** add `skills/leads/SKILL.md` to your project
rules or agent instructions. It is plain Markdown describing the endpoints,
credit rules and CSV format.

## 3. Try it without spending credits

Every endpoint has a free demo mirror under `/api/v2/demo/*` (no key needed,
responses carry `demo: true`):

```bash
curl -s https://api.leadmerlin.com/api/v2/demo/jobs/x/results | jq '.businesses[0]'
```

## n8n template

[`n8n/google-maps-leads-to-google-sheets.json`](n8n/google-maps-leads-to-google-sheets.json)
— search → poll until complete → paginate results → keep verified emails →
append/update a Google Sheet.

1. n8n → *Import from file* → pick the JSON.
2. Create a **Header Auth** credential named `LeadMerlin x-api-key` with name `x-api-key` and your key as value; select it on the three HTTP nodes.
3. In **Inputs**, keep `api_base` on `/api/v2/demo` for a free test run, then switch to `https://api.leadmerlin.com/api/v2`.
4. Pick your sheet in **Google Sheets**. Header row: `name, owner, email, email_status, phone, phone_type, website, city, rating, reviews, cms, gbp_url`.

Step-by-step guide: [leadmerlin.com/blog/n8n-google-maps-leads-to-google-sheets](https://leadmerlin.com/blog/n8n-google-maps-leads-to-google-sheets/?utm_source=github&utm_medium=skill-repo)

## Credits

1 credit per enriched business with a website. Listings without a website and
off-niche results are free; failed enrichments are refunded. Docs:
[leadmerlin.com/docs](https://leadmerlin.com/docs?utm_source=github&utm_medium=skill-repo).

## Responsible use

If you email or call the businesses you find, follow the laws that apply to
you (CAN-SPAM, TCPA, GDPR). See our
[CAN-SPAM checklist](https://leadmerlin.com/blog/can-spam-cold-email-local-businesses-checklist/?utm_source=github&utm_medium=skill-repo).

MIT licensed.
