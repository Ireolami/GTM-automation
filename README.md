GTM Lead-Gen & ICP Scoring Automation

An n8n workflow that discovers, enriches, scores, and routes SMEs across Nigeria and developed
African markets (Kenya, South Africa, Ghana, Egypt, Rwanda) that are high-intent buyers of
WhatsApp / voice AI agents.

Built by HMC Tech. Daily automated pipeline → scored leads in Supabase + Airtable, with a
ranked email digest of the day's best prospects.


What it does

Every day the workflow:


Scrapes candidate businesses from three sources — Instagram (Apify), Google Maps (Apify),
and LinkedIn/company data (Apify).
Normalizes every result to one common lead schema.
Deduplicates against Supabase (DB-level unique index — safe to re-run).
Enriches missing contacts: WhatsApp/email extraction, phone-code validation, buying signals.
Scores each lead with Claude against a weighted ICP rubric (0–100), returning strict JSON.
Persists survivors (score ≥ threshold) to Supabase + Airtable; routes the rest to an audit table.
Emails an HTML digest of Tier A + top B leads to smarworld25@gmail.com.



Why it exists

Nigerian and African SMEs are drowning in WhatsApp DMs and repetitive customer inquiries but rarely
have a dev team. That makes them ideal buyers for WhatsApp/voice AI agents. This pipeline finds them
at scale, scores intent, and hands sales a ranked list every morning — replacing manual prospecting.


Target buyers (ICP segments)

E-commerce / social sellers · real estate & PropTech · fintech & lending · health clinics & labs ·
edtech & training academies · logistics & last-mile · hospitality & short-lets · insurance & HMOs ·
professional services · B2B SaaS & agencies (channel partners) · forex/trading academies.

Country tiers: Tier 1 — Nigeria, Kenya, South Africa · Tier 2 — Ghana, Egypt, Rwanda.


Architecture at a glance

Schedule/Manual → Config
   → [Apify IG | Apify GMaps | Apify LinkedIn] → normalize
   → Merge → Dedup gate → Enrichment → Claude scoring → Filter
      ├─ pass → Supabase leads → Airtable CRM → Gmail digest
      └─ fail → Supabase rejected (audit)
   (every external call logs failures → Supabase run_log)

Full design in ARCHITECTURE.md.


Tech stack


n8n Cloud — orchestration
Apify — Instagram, Google Maps, and LinkedIn/company scrapers
Anthropic Claude (claude-sonnet-4-6) — weighted ICP scoring, JSON-only output
Supabase (Postgres) — source of truth (leads, rejected, run_log)
Airtable — working sales CRM
Gmail — daily digest delivery



Repository contents

FilePurposeREADME.mdThis file — project overview and quick start.ARCHITECTURE.mdFull system design: diagram, schema, rubric, storage, error handling.CLAUDE.mdOperating guide for any agent working in the repo + a living LESSONS log.workflow.jsonImportable n8n workflow (add when built).supabase.sqlDDL for leads, rejected, run_log + dedup index (add when built).


Quick start


Import workflow.json into n8n Cloud.
Apply the Supabase DDL (supabase.sql), including the unique dedup index.
Create the Airtable base and fields (see ARCHITECTURE.md §7).
Set credentials in n8n (by name — never inline):


   ANTHROPIC_API_KEY   APIFY_TOKEN   SUPABASE_URL   SUPABASE_SERVICE_ROLE_KEY
   AIRTABLE_API_KEY    AIRTABLE_BASE_ID   GMAIL_OAUTH


Configure tunables in the Config node: SEARCH_TERMS, TARGET_CITIES, PER_SOURCE_CAP,
SCORE_THRESHOLD, DIGEST_RECIPIENT, COUNTRY_TIERS.
Smoke-test: run the Manual trigger with PER_SOURCE_CAP = 5, confirm a lead lands in
Supabase and Airtable, then re-run and confirm zero duplicates.
Enable the daily Schedule.



Scoring rubric

SignalWeightWhatsApp Business number present20High DM/support-volume signal20ICP segment fit15Company size 5–100 (SME sweet spot)10Country tier10Active paid ads / growth signal10Repetitive-inquiry business nature10Reachable decision-maker5

Leads scoring ≥ 60 (configurable) are kept. Full output contract in ARCHITECTURE.md §6.


Operating principles


Idempotent — re-running a batch never creates duplicates (DB-level guarantee).
Fail loud, continue soft — external calls retry + continue-on-fail; failures log to run_log.
Behavior over config — a branch isn't done until it has produced a real row end-to-end.
No secrets in the graph — credentials by name; tunables in Config.
Claude learns from mistakes — every break is logged in CLAUDE.md §8 and load-bearing
lessons graduate into permanent Golden Rules.


Read CLAUDE.md before making any change.


Roadmap


 Ship workflow.json and supabase.sql.
 Add a phase-2 outreach workflow: feed Tier A leads into WhatsApp/cold-email sequences.
 Optional notify channels (Slack, WhatsApp alert) alongside the email digest.
 Weekly rejected-reason review feeding rubric tuning.



Maintained by HMC Tech. Digest recipient: smarworld25@gmail.com
