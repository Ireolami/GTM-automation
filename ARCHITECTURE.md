# ARCHITECTURE.md — GTM Lead-Gen & ICP Scoring Automation

> Automated pipeline that discovers, enriches, scores, and routes SMEs across Nigeria and
> developed African markets (Kenya, South Africa, Ghana, Egypt, Rwanda) that are high-intent
> buyers of WhatsApp / voice AI agents. Built on n8n Cloud.

---

## 1. System Overview

A daily-triggered n8n workflow pulls candidate businesses from three sources, normalizes them
to a common schema, deduplicates against Supabase, enriches missing contacts, scores each lead
with Claude against a weighted ICP rubric, persists survivors to Supabase + Airtable, and emails
a ranked digest of the day's best leads.

```
                          ┌─────────────┐
                          │  Schedule   │ (daily) + Manual trigger
                          └──────┬──────┘
                                 │
                          ┌──────▼──────┐
                          │ Config node │  search terms, cities, per-source caps
                          └──────┬──────┘
                                 │  (fan-out)
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│ Apify: IG     │        │ Apify: GMaps  │        │ Apify: LinkedIn│
│ scraper       │        │ scraper       │        │ / company      │
└───────┬───────┘        └───────┬───────┘        └───────┬───────┘
        │ normalize              │ normalize              │ normalize
        └────────────────────────┼────────────────────────┘
                                 ▼
                          ┌─────────────┐
                          │    Merge    │  common schema stream
                          └──────┬──────┘
                                 ▼
                          ┌─────────────┐
                          │ Dedup gate  │  query Supabase `leads` by norm key
                          └──────┬──────┘  (drop existing)
                                 ▼
                          ┌─────────────┐
                          │ Enrichment  │  WhatsApp/email extract, phone validate, signals
                          └──────┬──────┘
                                 ▼
                          ┌─────────────┐
                          │ Claude score│  Anthropic API, JSON-only, weighted rubric
                          └──────┬──────┘
                                 ▼
                          ┌─────────────┐
                          │   Filter    │  score >= threshold ?
                          └───┬─────┬───┘
                        pass  │     │  fail
                              ▼     ▼
                     ┌────────────┐ ┌──────────────┐
                     │ Supabase   │ │ Supabase     │
                     │ leads      │ │ rejected     │
                     │ (upsert)   │ │ (audit)      │
                     └─────┬──────┘ └──────────────┘
                           ▼
                     ┌────────────┐
                     │ Airtable   │  CRM upsert
                     │ CRM        │
                     └─────┬──────┘
                           ▼
                     ┌────────────┐
                     │ Gmail      │  HTML digest → smarworld25@gmail.com
                     │ digest     │
                     └────────────┘

  (every external call also logs failures → Supabase `run_log`)
```

---

## 2. Design Principles

1. **Idempotent + re-runnable.** The dedup gate + upserts guarantee that re-running the same day
   never double-inserts. Uniqueness is enforced at the DB level, not just in-flow.
2. **Behavioral over config trust.** A branch that "looks wired" is not proven until it has
   produced a row end-to-end. Verify by running, not by inspecting node config. (Hard-won lesson
   from prior Wa'se audits — dead branches that never executed in production.)
3. **Fail loud to a log, continue soft in the flow.** External calls (Apify, Anthropic, Airtable)
   use continue-on-fail + retry, and every failure writes to `run_log`. A single bad item never
   kills the batch.
4. **No secrets in the graph.** Credentials are referenced by name; search terms, caps, thresholds
   live in the Config node or env, never hardcoded in business-logic nodes.
5. **Separation of persist and notify.** Data lands in Supabase (source of truth) before Airtable
   (working CRM) before Gmail (human view). Each stage is independently replayable.

---

## 3. Data Sources

| Source | Actor | Targets | Key signals extracted |
|---|---|---|---|
| Instagram | Apify IG scraper | Social sellers, skincare, fashion, food, realtors | WhatsApp-in-bio, "DM to order", followers, link-in-bio catalog |
| Google Maps | Apify GMaps scraper | Clinics, labs, real estate, restaurants, logistics, academies | Phone, website, category, reviews volume, hours |
| LinkedIn/company | Apify LinkedIn scraper | Agencies, fintech, SaaS (channel-partner ICP) | Company size, decision-maker, industry |

Each source branch ends with a **normalize node** mapping raw output to the common schema (§5).

---

## 4. Target ICP Segments

Tagged on every lead; feeds the scoring rubric's segment-fit weight.

`ecommerce_social` · `real_estate_proptech` · `fintech_lending` · `health_clinic_lab` ·
`edtech_academy` · `logistics_delivery` · `hospitality_shortlet` · `insurance_hmo` ·
`professional_services` · `saas_agency_partner` · `forex_trading_academy`

Country tiers (scoring weight): **Tier 1** Nigeria, Kenya, South Africa · **Tier 2** Ghana, Egypt, Rwanda.

---

## 5. Common Lead Schema

Every source normalizes to this before merge:

```json
{
  "source": "instagram | gmaps | linkedin",
  "business_name": "string",
  "category": "string",
  "segment": "one of the ICP segment tags",
  "country": "NG | KE | ZA | GH | EG | RW",
  "city": "string",
  "website": "string | null",
  "phone": "string | null",
  "whatsapp": "string | null",
  "instagram": "string | null",
  "linkedin": "string | null",
  "followers": "int | null",
  "bio_text": "string | null",
  "has_whatsapp_business": "bool",
  "runs_paid_ads": "bool",
  "auto_reply_detected": "bool",
  "raw_json": "object (original payload for audit)"
}
```

---

## 6. Scoring (Claude node)

- **Model:** `claude-sonnet-4-6`, `max_tokens: 1000`, JSON-only output (no markdown fences).
- **Weighted rubric (0–100):**

| Signal | Weight |
|---|---|
| WhatsApp Business number present | 20 |
| High DM/support-volume signal | 20 |
| ICP segment fit | 15 |
| Company size 5–100 (SME sweet spot) | 10 |
| Country tier | 10 |
| Active paid ads / growth signal | 10 |
| Repetitive-inquiry business nature | 10 |
| Reachable decision-maker | 5 |

- **Output contract (strict JSON):**

```json
{
  "score": 0,
  "tier": "A | B | C",
  "segment": "string",
  "top_signals": ["string"],
  "suggested_hook": "string",
  "decision_maker_guess": "string",
  "disqualify_reason": "string | null"
}
```

- **Filter:** keep `score >= 60` (configurable). Below → `rejected` table.
- **Parsing:** strip any accidental fences, `JSON.parse` in try/catch, on failure log to `run_log`
  and route the item to `rejected` with reason `score_parse_error` — never let it crash the batch.

---

## 7. Storage

### Supabase (source of truth)
- `leads` — surviving, scored leads. Unique index enforces dedup.
- `rejected` — disqualified + parse-error items, for audit and rubric tuning.
- `run_log` — per-node failures: `{run_id, node, error, item_key, created_at}`.

### Airtable (working CRM)
- One base, `Leads` table. Fields: Business, Segment, Country, City, Score, Tier, WhatsApp,
  Website, Instagram, Top Signals, Suggested Hook, Decision Maker, Status (`New` default), Source.
- Upsert keyed on a stable external id (`supabase_lead_id`) to avoid duplicate CRM rows.

### Gmail (human view)
- Daily HTML digest → `smarworld25@gmail.com`: Tier A + top B, table layout, plus counts per
  source and per country and the `run_log` error count for the day.

---

## 8. Dedup Strategy

Normalized key = `lower(trim(business_name)) || '|' || country`, with secondary match on
`phone` and `website` when present. Enforced by a Supabase unique index so concurrent/re-runs
cannot double-insert. The in-flow dedup gate is an optimization; the DB constraint is the guarantee.

---

## 9. Error Handling & Observability

- Apify, Anthropic, Airtable, Gmail nodes: **retry (3x, backoff) + continue-on-fail**.
- Every catch path writes to `run_log`.
- Digest surfaces the day's error count so silent degradation is visible.
- A weekly review of `rejected` reasons feeds rubric tuning (see LESSONS in CLAUDE.md).

---

## 10. Environment / Secrets

Referenced by credential name in n8n; never inline.

```
ANTHROPIC_API_KEY
APIFY_TOKEN
SUPABASE_URL
SUPABASE_SERVICE_ROLE_KEY
AIRTABLE_API_KEY
AIRTABLE_BASE_ID
GMAIL_OAUTH (n8n credential)
```

Config node (non-secret, tunable): `SEARCH_TERMS`, `TARGET_CITIES`, `PER_SOURCE_CAP`,
`SCORE_THRESHOLD`, `DIGEST_RECIPIENT`, `COUNTRY_TIERS`.

---

## 11. Extension Points

- Add a source branch by conforming to the common schema + merge — no downstream change needed.
- Swap the notify channel (Slack, WhatsApp alert) by adding a parallel node after digest aggregation.
- Feed Tier A leads directly into an outreach sequence (cold email / WhatsApp) as a phase-2 workflow.
